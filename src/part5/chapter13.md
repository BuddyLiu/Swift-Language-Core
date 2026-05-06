# 第13章 闭包与捕获语义

闭包（Closure）是 Swift 中最为强大也最为常用的特性之一。简单来说，闭包是一个可以捕获并存储其上下文中的常量和变量的可调用代码块。Swift 中的函数本身就是一种特殊的闭包（有名字的闭包），而闭包则可以被理解为"匿名函数"的升级版——它不仅能被传递和使用，还能"记住"它被创建时的环境。

理解闭包及其捕获语义，是掌握 Swift 内存管理的核心前提。本章将从语法糖入手，逐步深入到逃逸语义、自动闭包、捕获列表以及内存安全等关键主题。

## 13.1 闭包的语法糖：尾随闭包、简写 $0

Swift 的闭包语法经历过多轮打磨，最终形成了一套极为简洁的表达方式。这些"语法糖"让闭包在代码中看起来格外轻量。

### 基础语法

一个完整的闭包表达式长这样：

```swift
let closure: (Int, Int) -> Int = { (a: Int, b: Int) -> Int in
    return a + b
}
```

花括号包裹整体，`in` 关键字分隔参数/返回值与函数体。但多数情况下，我们不需要写这么完整。

### 类型推断

当闭包作为参数传递时，Swift 编译器通常已经知道参数和返回值的类型，因此可以省略：

```swift
// 完整版
let result1 = numbers.map({ (element: Int) -> Int in
    return element * 2
})

// 类型推断简写
let result2 = numbers.map({ element in
    return element * 2
})
```

编译器从 `map` 的泛型签名推断出 `element` 是 `Int`，返回值也是 `Int`，因此中间的显式类型标注都可以省略。

### 隐式返回

如果闭包体只有一行表达式，可以省略 `return`：

```swift
let doubled = numbers.map({ element in element * 2 })
```

单行表达式的值自动成为闭包的返回值。

### 简写参数名 $0, $1, $2, ...

这是最常用的语法糖。Swift 为闭包参数提供了自动的简写名称，`$0` 表示第一个参数，`$1` 表示第二个，以此类推：

```swift
let sorted = names.sorted { $0 > $1 }

let doubled = numbers.map { $0 * 2 }

let filtered = numbers.filter { $0.isMultiple(of: 3) }

// 两个参数时也很常见
let zipped = zip(names, scores).map { "\($0): \($1)" }
```

使用 `$0` 后，参数列表（`in` 之前的部分）可以完全省略。

### 尾随闭包（Trailing Closure）

当闭包是函数的最后一个参数时，可以将其写在函数调用的圆括号之外：

```swift
// 非尾随
let result = performOperation(10, 20, operation: { $0 + $1 })

// 尾随闭包
let result2 = performOperation(10, 20) { $0 + $1 }
```

如果函数只有一个参数且是闭包，圆括号都可以省略：

```swift
let numbers = [3, 1, 4, 1, 5, 9, 2]

// sorted(by:) 接受一个闭包参数
let sorted = numbers.sorted { $0 < $1 }

// 这比 C 语言风格的回调或 block 语法简洁得多
```

Swift 还支持多个尾随闭包，这在 SwiftUI 中非常常见：

```swift
UIView.animate(withDuration: 0.3) {
    // 动画内容
} completion: { finished in
    // 完成回调
}
```

### 操作符函数

Swift 中的操作符本身也是函数，因此可以直接传递：

```swift
let ascending = numbers.sorted(by: <)   // 等价于 { $0 < $1 }
let descending = numbers.sorted(by: >)  // 等价于 { $0 > $1 }
let sum = [1, 2, 3].reduce(0, +)       // 等价于 { $0 + $1 }
```

这种写法让代码读起来几乎像自然语言。

## 13.2 逃逸闭包 @escaping 与非逃逸闭包

闭包有一个重要的分类维度：逃逸（escaping）与非逃逸（non-escaping）。

### 非逃逸闭包（默认）

默认情况下，函数参数中的闭包是非逃逸的，意味着闭包的生命周期不会超出函数调用的范围：

```swift
func performOperation(_ value: Int, using closure: (Int) -> Int) -> Int {
    // closure 只在函数体内部被调用
    return closure(value)
}
```

非逃逸闭包有几个重要的编译器优化优势：
- 编译器可以省略闭包对象的引用计数操作，提升性能
- 闭包捕获的变量不需要 `self` 显式标注
- 编译器可以优化内存分配，甚至将闭包内联展开

### 逃逸闭包 @escaping

当闭包在函数返回之后仍可能被调用时，需要用 `@escaping` 显式标记：

```swift
class NetworkService {
    var completion: ((Data?) -> Void)?

    func fetchData(completion: @escaping (Data?) -> Void) {
        // 闭包被存储到属性中，可能在调用返回后很久才执行
        self.completion = completion
    }

    func fetchAsync(completion: @escaping (Data?) -> Void) {
        DispatchQueue.global().async {
            // 异步队列中的闭包也会逃逸
            let data = try? Data(contentsOf: url)
            completion(data)
        }
    }
}
```

逃逸闭包需要满足两个条件：
1. 闭包必须在函数返回后仍然存活（被存储、被异步执行等）
2. 闭包必须显式标记 `@escaping`

### 逃逸闭包的语义差异

逃逸闭包与非逃逸闭包有几个关键差异：

```swift
class MyClass {
    var x = 10

    func nonEscapingExample() {
        // 非逃逸闭包中不需要写 self
        perform { x += 1 }
    }

    func escapingExample() {
        // 逃逸闭包中必须显式使用 self
        performAsync { self.x += 1 }
    }

    func perform(_ block: () -> Void) {
        block()
    }

    func performAsync(_ block: @escaping () -> Void) {
        DispatchQueue.main.async { block() }
    }
}
```

这是因为非逃逸闭包在函数返回前就会执行完毕，闭包不可能比 `self` 存活更久；而逃逸闭包可能会在 `self` 被释放后调用，因此需要显式声明对 `self` 的捕获意图。

### 从 Swift 5.3 开始的隐式 self

在 Swift 5.3 及之后，某些情况下逃逸闭包可以省略 `self`：

```swift
struct Counter {
    var count = 0

    mutating func increment() {
        count += 1
    }
}

// 值类型（结构体、枚举）的逃逸闭包不能捕获 self
// 因为值类型在闭包中是被复制的，修改不影响原值
```

但对于引用类型（类）的逃逸闭包，`self` 仍然是必需的。

## 13.3 自动闭包 @autoclosure：延迟求值

`@autoclosure` 是一个特殊的属性，它将一个表达式自动包装成闭包，从而实现延迟求值。

### 基本用法

```swift
func logIfTrue(_ condition: @autoclosure () -> Bool) {
    if condition {
        print("条件为真")
    }
}

// 调用时看起来像传递普通值
logIfTrue(3 > 2)
```

`3 > 2` 本应是 `Bool` 类型的值，但由于参数被标记为 `@autoclosure () -> Bool`，编译器自动将其包装成一个闭包 `{ 3 > 2 }`。

### 延迟求值的意义

如果没有 `@autoclosure`，表达式在传入函数之前就已经被求值了。对于代价高昂的操作，延迟求值可以避免不必要的计算：

```swift
func expensiveCheck() -> Bool {
    print("执行了高代价检查")
    return true
}

func logIfTrue(_ condition: @autoclosure () -> Bool) {
    // 这里才真正求值
    if condition {
        print("条件为真")
    }
}

// expensiveCheck() 不会立即执行，只有 logIfTrue 内部用到时才执行
logIfTrue(expensiveCheck())
```

### 短路求值模拟

Swift 中的逻辑操作符 `&&` 和 `||` 支持短路求值，其底层就是利用了自动闭包：

```swift
// Swift 标准库中 && 的简化实现
static func && (left: Bool, right: @autoclosure () throws -> Bool) rethrows -> Bool {
    if left {
        return try right()
    }
    return false
}
```

如果 `left` 已经是 `false`，右侧表达式根本不会被求值。这正是自动闭包实现的延迟计算。

### 自动闭包与逃逸

自动闭包默认是非逃逸的。如果需要逃逸，需要同时使用 `@autoclosure @escaping`：

```swift
var handlers: [() -> Bool] = []

func addHandler(_ handler: @autoclosure @escaping () -> Bool) {
    handlers.append(handler)
}

addHandler(5 > 3)
addHandler(2 > 1)
// 延迟到后续某个时刻统一执行
for handler in handlers {
    print(handler())  // true, true
}
```

### 实际应用场景

`@autoclosure` 最常见的场景是断言（assert）：

```swift
// Swift 标准库的 assert 定义
func assert(_ condition: @autoclosure () -> Bool,
            _ message: @autoclosure () -> String = String()) {
    // Release 模式下 condition 不会被求值
    // message 也不会被计算
}
```

在 Release 构建中，`assert` 的闭包体不会被执行，这意味着传入的复杂表达式不会产生任何性能开销——这正是自动闭包延迟求值的威力。

## 13.4 捕获列表与内存安全

闭包可以"捕获"其作用域中的变量和常量。捕获机制是闭包最强大的能力，但如果使用不当，也会成为内存问题的根源。

### 值捕获

闭包捕获值类型时，默认会创建一份副本：

```swift
func makeIncrementer(step: Int) -> () -> Int {
    var total = 0
    return {
        total += step  // total 和 step 被捕获
        return total
    }
}

let incrementByTwo = makeIncrementer(step: 2)
print(incrementByTwo())  // 2
print(incrementByTwo())  // 4
print(incrementByTwo())  // 6

// 每个闭包有自己独立的捕获副本
let incrementByTen = makeIncrementer(step: 10)
print(incrementByTen())  // 10
```

注意这里的 `total` 是栈上的局部变量，但闭包将其捕获后，它会被"提升"到堆上，以确保闭包和调用者的安全共存。

### 引用捕获

引用类型（类实例）被闭包捕获时，闭包会持有该对象的强引用：

```swift
class Counter {
    var count = 0
    func increment() { count += 1 }
}

func createClosure() -> () -> Void {
    let counter = Counter()
    return {
        counter.increment()  // 强引用 counter
        print(counter.count)
    }
}

let closure = createClosure()
closure()  // 1
closure()  // 2
// counter 不会被释放，因为闭包仍然持有它
```

### 捕获列表（Capture List）

捕获列表允许我们控制闭包如何捕获变量。它在闭包的开头、`in` 关键字之前使用方括号声明：

```swift
var value = 10
let closure = { [value] in
    // 这里的 value 是捕获时的副本
    print(value)
}
value = 20
closure()  // 打印 10，而不是 20
```

捕获列表在闭包创建时生成独立的副本，后续外部变量的变化不会影响闭包内部。

### 捕获列表对引用类型的作用

对于引用类型，捕获列表可以指定捕获方式为 `weak` 或 `unowned`：

```swift
class MyClass {
    var value = 42
}

var obj: MyClass? = MyClass()

// 强捕获
let strongCapture = {
    print(obj?.value ?? 0)  // 闭包持有 obj 的强引用
}

// 弱捕获
let weakCapture = { [weak obj] in
    print(obj?.value ?? 0)  // 闭包持有 obj 的弱引用
}

obj = nil
strongCapture()  // nil（但 obj 不会被释放，因为闭包还持有着）
weakCapture()    // 0（obj 已释放，安全访问）
```

### 捕获列表的语法细节

捕获列表可以捕获多个变量，也可以结合重命名：

```swift
var a = 1, b = 2
let closure = { [a, b] in
    // 捕获 a 和 b 的副本
    print(a, b)
}

// 重命名捕获
let renamed = { [capturedA = a, capturedB = b] in
    print(capturedA, capturedB)
}

// 混合使用
class Controller {
    var data: [String] = []
    func setup() {
        // 捕获 self 为弱引用，同时捕获 data 的副本
        let closure = { [weak self, data = self.data] in
            guard let self else { return }
            // 使用 data 的副本和弱引用的 self
        }
    }
}
```

## 13.5 [weak self] 的正确使用

`[weak self]` 是 iOS/macOS 开发中最常用的内存管理模式之一。它存在于几乎每个异步回调、闭包存储的场景中。

### 为什么需要 weak self？

当类的实例持有一个闭包，而闭包又捕获了该实例的 `self` 时，就会形成强引用循环（retain cycle）：

```swift
class NetworkManager {
    var onComplete: (() -> Void)?
    var data: Data?

    func startRequest() {
        // 错误：形成强引用循环
        onComplete = {
            // 闭包持有 self（强引用）
            // self 持有 onComplete（强引用）
            self?.processData()
        }
    }

    func processData() {
        // 处理数据...
    }
}
```

在这个例子中，`NetworkManager` 持有 `onComplete` 属性，而闭包又捕获了 `self`，两者互相强引用，导致对象无法释放。

### 使用 [weak self] 打破循环

```swift
class NetworkManager {
    var onComplete: (() -> Void)?
    var data: Data?

    func startRequest() {
        onComplete = { [weak self] in
            guard let self else { return }
            // 这里的 self 是弱引用，不增加引用计数
            self.processData()
        }
    }

    func processData() {
        // 处理数据...
    }
}
```

使用 `[weak self]` 后，闭包不再强持有 `self`。当 `NetworkManager` 被其他所有者释放后，`self` 变为 `nil`，闭包中的 `guard let self` 会安全退出。

### weak self 与 unowned self 的选择

Swift 提供了两种避免强引用的方式：

```swift
// weak：可选类型，当对象释放后自动变为 nil
let closure1 = { [weak self] in
    guard let self else { return }
    self.doSomething()
}

// unowned：非可选类型，假定对象不会先于闭包释放
let closure2 = { [unowned self] in
    self.doSomething()  // 如果 self 已释放，这里会崩溃
}
```

**选择原则**：
- `weak`：当你不能确定 `self` 的生命周期是否长于闭包时，使用 `weak`，它是安全的
- `unowned`：当你确定 `self` 一定比闭包存活更久时，可以使用 `unowned`，避免可选绑定的开销

`weak` 更安全，应该是默认选择。`unowned` 适用于特定场景，如闭包的生命周期严格受限于 `self`。

### 常见模式和最佳实践

**模式一：guard let self**

```swift
class ViewController: UIViewController {
    func loadData() {
        api.fetchData { [weak self] result in
            guard let self else { return }
            // 这里 self 已被解包，后续调用不需要写 ?
            self.updateUI(with: result)
        }
    }
}
```

**模式二：处理多个闭包**

```swift
class DownloadManager {
    var progressHandler: ((Double) -> Void)?
    var completionHandler: (() -> Void)?

    func download() {
        URLSession.shared.dataTask(with: url) { [weak self] data, _, error in
            guard let self else { return }
            self.progressHandler = { [weak self] progress in
                self?.updateProgressBar(progress)
            }
            self.completionHandler = { [weak self] in
                self?.showCompletion()
            }
        }
    }
}
```

**模式三：链式闭包**

```swift
class DataProcessor {
    func process() {
        firstStep { [weak self] in
            guard let self else { return }
            self.secondStep { [weak self] in
                guard let self else { return }
                self.thirdStep()
            }
        }
    }
}
```

注意每一层闭包都需要独立的 `[weak self]` 捕获。

### 误区提醒

**误区一：所有闭包都用 weak self**

非逃逸闭包不需要 `[weak self]`，因为它们不会形成引用循环。滥用 `[weak self]` 会让代码变得不必要的复杂。

**误区二：忘记用 weak self 但以为用了**

```swift
// 错误：闭包里写了 self，但捕获列表中没有 weak
onComplete = {
    self.processData()  // 强引用！
}
```

只要在捕获列表中没有声明 `[weak self]`，即使闭包体里写了 `self`，仍然是强引用。

**误区三：在值类型中使用 weak self**

结构体和枚举是值类型，不存在引用循环的问题。`[weak self]` 只适用于类实例，对值类型使用会编译错误。

## 本章小结

闭包是 Swift 中表达力和复杂度兼具的核心特性。从简洁的语法糖（`$0`、尾随闭包）到精细的内存控制（捕获列表、`@escaping`、`@autoclosure`），闭包在提供强大的抽象能力的同时，也要求开发者对内存管理有清晰的认识。掌握闭包的捕获语义和 `[weak self]` 的正确用法，是写出高性能、无内存泄漏的 Swift 代码的必备技能。
