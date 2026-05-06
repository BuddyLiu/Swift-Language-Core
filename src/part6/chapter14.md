# 第14章 自动引用计数深层解析

自动引用计数（Automatic Reference Counting，简称 ARC）是 Swift 管理引用类型内存的核心机制。与 GC（垃圾回收）语言不同，Swift 的 ARC 在编译期插入引用计数的增减指令，不依赖运行时扫描，因此具有可预测的性能特征。

本章将从引用计数的底层原理入手，深入探讨强引用循环的检测与解决方案，分析闭包捕获对 ARC 的影响，最后对比值类型和引用类型在内存管理上的本质差异。

## 14.1 引用计数的原理

### 对象的生命周期

在 ARC 机制下，每个类实例都有一个关联的引用计数（Reference Count）。当计数变为 0 时，该实例立即被释放。

```swift
class Person {
    let name: String
    init(name: String) {
        self.name = name
        print("\(name) 被创建")
    }
    deinit {
        print("\(name) 被释放")
    }
}

var person1: Person? = Person(name: "Alice")  // 引用计数 = 1
var person2 = person1                          // 引用计数 = 2
person1 = nil                                  // 引用计数 = 1
person2 = nil                                  // 引用计数 = 0 → 释放
```

ARC 的工作机制可以概括为三个操作：

- **retain**：引用计数 +1（当一个新的强引用指向该对象时）
- **release**：引用计数 -1（当一个强引用被移除时）
- **dealloc**：当引用计数为 0 时，对象被释放，内存被回收

### 引用计数的存储

每个类实例的引用计数存储在实例的**内存头部**，与实例本身的元数据（如 isa 指针）放在一起。在 Swift 中，对象的堆内存布局大致如下：

```
┌─────────────────────────┐
│  isa（指向元数据）        │
├─────────────────────────┤
│  引用计数 (refCount)      │
├─────────────────────────┤
│  属性存储...              │
├─────────────────────────┤
│  ...                     │
└─────────────────────────┘
```

Swift 使用"引用计数边带"（side table）技术进行优化。当引用计数超过一定阈值时，会创建一个 side table 来存储扩展的计数信息，避免频繁的原子操作。

### ARC 的编译时插入

```swift
// 源码
func use(person: Person) {
    let p = person
    print(p.name)
}

// 编译器插入 ARC 指令后的伪代码
func use(person: Person) {
    retain(person)       // 引用计数 +1
    let p = person
    print(p.name)
    release(p)           // 引用计数 -1
}
```

编译器通过编译器分析（编译器在 SIL 层面进行优化），可以消除冗余的 retain/release 操作。

### 手动管理 vs ARC

尽管 ARC 是自动的，但理解其底层操作有助于写出更高效的代码：

```swift
// 不必要的临时变量会增加 ARC 操作
func process(items: [Person]) {
    // 循环中每次迭代都可能有额外的 retain/release
    for i in 0..<items.count {
        let item = items[i]  // retain
        print(item.name)
        // release
    }
}

// 使用 for-in 语法，编译器可以更好地优化
func processBetter(items: [Person]) {
    for item in items {  // 编译器可以合并 retain/release
        print(item.name)
    }
}
```

## 14.2 强引用循环的发现与解决

### 什么是强引用循环

当两个或多个对象互相持有强引用时，就形成了强引用循环，导致对象无法被释放：

```swift
class Department {
    let name: String
    var manager: Manager?  // Department 持有 Manager
    init(name: String) { self.name = name }
    deinit { print("\(name) 部门被释放") }
}

class Manager {
    let name: String
    var department: Department?  // Manager 持有 Department
    init(name: String) { self.name = name }
    deinit { print("\(name) 经理被释放") }
}

var dept: Department? = Department(name: "技术部")
var mgr: Manager? = Manager(name: "Bob")

dept!.manager = mgr
mgr!.department = dept

// 即使将外部引用置 nil
dept = nil  // 不会触发 deinit
mgr = nil  // 不会触发 deinit
// 两个对象仍然互相持有，形成"孤岛"
```

### 发现强引用循环的方法

**方法一：deinit 打印**

在类的 `deinit` 中添加日志是最直接的检测方式：

```swift
class MyViewController: UIViewController {
    deinit {
        print("MyViewController 被释放")
    }
}
```

如果离开页面后没有看到打印，说明存在强引用循环。

**方法二：内存图调试器**

Xcode 提供了强大的内存图调试器（Memory Graph Debugger）：
1. 在运行时暂停程序
2. 点击 Debug Memory Graph 按钮
3. 查看左侧的实例列表，寻找不应该存活的对象
4. 选择可疑对象，查看其引用链

**方法三：Leaks 检测**

使用 Instruments 的 Leaks 工具可以自动检测强引用循环。

**方法四：弱引用标记**

在怀疑有循环的地方，尝试将某些属性改为 `weak`，观察 deinit 是否被触发。

### 解决强引用循环

**使用弱引用（weak）**

```swift
class Manager {
    let name: String
    weak var department: Department?  // 弱引用，不增加引用计数
    init(name: String) { self.name = name }
    deinit { print("\(name) 经理被释放") }
}
```

`weak` 引用必须是可选类型，因为对象释放后会自动变为 `nil`。

**使用无主引用（unowned）**

当确定一个对象的生命周期不会超过另一个对象时，可以使用 `unowned`：

```swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) { self.name = name }
    deinit { print("\(name) 被释放") }
}

class CreditCard {
    let number: String
    unowned let customer: Customer  // 信用卡必然属于某位客户
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { print("信用卡 \(number) 被释放") }
}
```

`unowned` 是非可选类型，访问已释放的 `unowned` 引用会导致崩溃。

### 判断原则

```
┌─────────────────────────────────────────────────────┐
│ 弱引用选择原则：                                       │
│                                                       │
│ 可选属性？         → weak                              │
│ 非可选且生命周期不重叠？ → unowned                      │
│ 两个对象寿命相当？    → 其中一个设 weak                  │
│ 有明显的"从属"关系？  → 从属方用 unowned 引用属主方      │
└─────────────────────────────────────────────────────┘
```

## 14.3 闭包捕获与 ARC

### 闭包捕获引用类型的 ARC 行为

当闭包捕获一个引用类型的变量时，闭包会持有该变量的强引用（除非使用捕获列表指定 `weak` 或 `unowned`）：

```swift
class Task {
    let id: Int
    init(_ id: Int) { self.id = id; print("Task \(id) 创建") }
    deinit { print("Task \(id) 释放") }
}

var closure: (() -> Void)? = {
    let task = Task(1)
    // 闭包执行完毕后，task 被释放
}

closure?()
closure?()
// 每次调用都是独立创建和释放
```

但如果闭包捕获了外部引用类型变量：

```swift
var task: Task? = Task(1)
let closure = {
    print(task?.id ?? 0)  // 闭包强引用 task
}
task = nil  // task 不会被释放，因为闭包还持有它
closure()   // 打印 nil
```

只有闭包本身被释放后，`task` 才能真正释放。

### 闭包引起的强引用循环

当类实例持有一个闭包，而闭包又捕获了该实例时，形成循环：

```swift
class HTMLElement {
    let name: String
    let text: String?

    // 这是一个 lazy 属性，其闭包会捕获 self
    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }

    deinit {
        print("\(name) 被释放")
    }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello")
paragraph!.asHTML()
paragraph = nil  // 不会释放，asHTML 闭包持有 self
```

### 闭包捕获列表的 ARC 影响

使用捕获列表解决闭包引起的强引用循环：

```swift
class HTMLElement {
    let name: String
    let text: String?

    lazy var asHTML: () -> String = { [weak self] in
        guard let self else { return "" }
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }

    deinit {
        print("\(name) 被释放")
    }
}
```

### 逃逸闭包的生命周期管理

异步操作中的闭包逃逸尤其需要关注 ARC 问题：

```swift
class ImageLoader {
    func loadImage(from url: URL, completion: @escaping (UIImage?) -> Void) {
        URLSession.shared.dataTask(with: url) { [weak self] data, _, error in
            guard let self else { return }
            // 在子线程中处理数据
            let image = self.processData(data)
            DispatchQueue.main.async {
                // 回到主线程回调
                completion(image)
            }
        }.resume()
    }

    private func processData(_ data: Data?) -> UIImage? {
        // 处理图片数据
        return nil
    }
}
```

每一层异步闭包都可能形成强引用循环，需要在每层闭包中考虑使用 `[weak self]`。

## 14.4 值类型为何不需要引用计数

### 值类型的本质：复制

值类型（结构体、枚举、元组）在赋值、传参时会被复制，而不是建立引用关系：

```swift
struct Point {
    var x: Double
    var y: Double
}

var p1 = Point(x: 1, y: 2)
var p2 = p1      // 复制：p2 是 p1 的独立副本
p2.x = 100       // 修改 p2 不影响 p1
print(p1.x)      // 1
```

每个值类型实例都有独立的生命周期，不存在多个变量指向同一块内存的情况（不考虑写时复制优化），因此不需要引用计数来跟踪共享状态。

### 值类型存储在栈上的传统模型

值类型的主要内存来源是栈（stack）：

```swift
func draw() {
    let point = Point(x: 10, y: 20)  // 在栈上分配
    let line = Line(start: point, end: point)  // 复制 point
    // 函数结束时，point 和 line 自动出栈
}
```

栈上的内存分配和释放是 O(1) 的，只需要移动栈指针，不需要引用计数的 retain/release 操作。

### 值类型的逃逸

值类型也可能"逃逸"到堆上，例如被闭包捕获或作为类的属性：

```swift
func makeClosure() -> () -> Void {
    var point = Point(x: 0, y: 0)  // 栈上的值
    return {
        point.x += 1  // point 被闭包捕获，提升到堆上
        print(point)
    }
}
```

即使值类型被提升到堆上，它仍然是值语义：修改其中一个副本不会影响其他副本。值类型的"值语义"保证了内存安全，这才是根本原因——引用计数是为了处理"共享"问题，而值类型从根本上避免了共享。

### 写时复制（Copy-on-Write）

某些值类型（如 `Array`、`Dictionary`、`Set`）内部可能持有堆上的缓冲区，但它们对外呈现值语义。它们使用写时复制技术来优化性能：

```swift
var array1 = [1, 2, 3, 4, 5]  // 内部缓冲区在堆上
var array2 = array1            // 共享同一缓冲区（此时引用计数为 2）

array2.append(6)               // 检测到引用计数 > 1，创建副本
```

写时复制内部确实使用了引用计数来判断是否需要复制，但这个引用计数是缓冲区内部的实现细节，对外部 API 来说，值类型的行为仍然是"不需要引用计数"的。

## 14.5 编译器优化：堆栈提升

### 什么是堆栈提升

堆栈提升（Stack Promotion）是 Swift 编译器的一项重要优化：如果编译器能证明某个堆上分配的对象不会逃逸出当前作用域，就将其改为在栈上分配。

```swift
// 编译器可能将这个临时 Person 提升到栈上
func greet(name: String) {
    let person = Person(name: name)  // 可能栈分配而非堆分配
    print("Hello, \(person.name)")
}  // 栈自动清理，不需要 release
```

### 编译器分析的依据

编译器通过 SIL（Swift Intermediate Language）级别的分析来决定是否进行堆栈提升：

```swift
// 可提升：对象不会逃逸
func canBePromoted() {
    let obj = MyClass()
    obj.doSomething()
    // obj 在函数结束时不会被外部持有
}

// 不可提升：对象返回出去了
func cannotBePromoted() -> MyClass {
    let obj = MyClass()
    return obj  // 逃逸了
}

// 不可提升：对象被存储到全局或属性
var global: MyClass?
func alsoCannotBePromoted() {
    let obj = MyClass()
    global = obj  // 逃逸到全局
}
```

### 闭包捕获与堆栈提升

闭包捕获会影响堆栈提升的决策：

```swift
// 非逃逸闭包中的捕获，对象可能被提升
func withNonEscaping(closure: () -> Void) {
    let obj = MyClass()
    closure()  // 闭包立即执行，结束后 obj 不再需要
}

// 逃逸闭包中的捕获，对象不能提升
func withEscaping(closure: @escaping () -> Void) {
    let obj = MyClass()
    closure = { obj.doSomething() }
    closure()  // 闭包逃逸了，obj 必须在堆上
}
```

### ARC 优化：retain/release 消除

编译器还进行多种 ARC 优化来减少不必要的引用计数操作：

**优化一：不相关的路径合并**

```swift
// 源码
func foo(x: MyClass) {
    let y = x
    print(y)
}

// 优化后（ARC 注入后可能消除冗余的 retain/release）
func foo$(x: MyClass) {
    let y = x  // 不需要 retain，因为函数参数已经持有强引用
    print(y)
    // 不需要 release，调用者负责
}
```

**优化二：循环中的 ARC 操作提升**

```swift
// 将 retain/release 移出循环体
for item in collection {
    // 编译器可能会将 retain 提到循环外，release 放到循环后
    process(item)
}
```

### 实际性能影响

堆栈提升和 ARC 优化在实际应用中的性能提升非常显著。在微基准测试中，优化后的代码可能比未优化的快数倍。开发者可以通过以下方式帮助编译器更好地优化：

1. 使用非逃逸闭包而非逃逸闭包
2. 避免不必要的闭包嵌套
3. 让值类型尽可能在栈上工作
4. 使用 Instruments 分析 retain/release 的热点

## 本章小结

ARC 是 Swift 内存管理的基石。理解引用计数的底层机制——对象的生命周期、闭包捕获的 ARC 影响、值类型与引用类型的本质差异——对于写出高性能且无内存泄漏的 Swift 代码至关重要。编译器优化（堆栈提升、retain/release 消除）在幕后做了大量工作，但开发者对捕获列表和 `weak self` 的正确使用，仍是保证内存安全的最重要防线。
