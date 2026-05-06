# 第7章 结构体与枚举：一等公民值类型

在 Swift 的现代化设计理念中，结构体和枚举不再是从属于类的二等公民，而是与类并驾齐驱的核心类型。它们承载着 Swift 最重要的设计哲学之一——**值语义**。理解并善用结构体与枚举，是写出安全、高效、可预测代码的关键。

## 7.1 结构体：不可变设计的天然载体

Swift 中的结构体（`struct`）远比 C 语言中的结构体强大。它可以定义属性、方法、构造器，甚至遵循协议和扩展——几乎拥有类的一切能力，唯独缺少继承。

```swift
struct Point {
    var x: Double
    var y: Double
    
    func distance(to other: Point) -> Double {
        let dx = x - other.x
        let dy = y - other.y
        return sqrt(dx * dx + dy * dy)
    }
}

let p1 = Point(x: 0, y: 0)
let p2 = Point(x: 3, y: 4)
print(p1.distance(to: p2)) // 5.0
```

结构体的**不可变设计**体现在两个层面。首先，Swift 会对结构体自动生成**成员逐一构造器**（Memberwise Initializer），无需手动编写：

```swift
// 不需要显式定义 init(x:y:)
let p = Point(x: 10, y: 20)
```

其次，结构体实例被声明为 `let` 后，其所有属性都不可变。这是因为结构体实例本身就是它的全部数据——没有引用间接层。这种特性使得结构体天然适合"值"的建模，例如坐标、范围、颜色、金额等。

```swift
struct Range {
    let start: Int
    let end: Int
    
    var length: Int {
        end - start
    }
    
    func contains(_ value: Int) -> Bool {
        value >= start && value < end
    }
}

// 即使 start 和 end 是 let 属性，计算属性仍然可以被访问
let range = Range(start: 0, end: 10)
print(range.contains(5))  // true
```

**何时使用结构体？** 苹果官方指南给出了明确建议：当类型符合以下一个或多个条件时，优先选择结构体：
- 主要目的是封装少量简单的数据值
- 实例被赋值或传递时，需要被拷贝而非引用
- 不需要继承其他类型的属性或行为

## 7.2 值语义与拷贝行为

值语义是结构体最本质的特征。当结构体实例被赋值给新变量或传递给函数时，Swift 会创建一份**独立的拷贝**。

```swift
struct Person {
    var name: String
    var age: Int
}

var person1 = Person(name: "Alice", age: 30)
var person2 = person1  // 完全拷贝

person2.name = "Bob"

print(person1.name)  // "Alice" — 不受影响
print(person2.name)  // "Bob"
```

这一行为与引用类型形成鲜明对比。类是引用类型，多个变量可以指向同一个实例：

```swift
class PersonClass {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

let c1 = PersonClass(name: "Alice", age: 30)
let c2 = c1  // 共享同一个实例

c2.name = "Bob"
print(c1.name)  // "Bob" — 被修改了！
```

值语义带来的核心优势是**数据隔离**。在多线程环境中，不同线程各自拥有独立拷贝，不会发生数据竞争。这就是为什么 Swift 标准库中几乎所有基础类型（`String`、`Array`、`Dictionary`、`Int`、`Double` 等）都是结构体。

深入理解，String 在 Objective-C 时代是引用类型（`NSString`），但在 Swift 中被重新设计为值类型。这一改变极大地提升了代码的安全性：

```swift
func appendString(_ str: String) -> String {
    var copy = str
    copy.append("!")
    return copy
}

let original = "Hello"
let modified = appendString(original)
print(original)  // "Hello" — 安全
```

## 7.3 写时复制（Copy-on-Write）：兼顾性能与安全

值语义虽好，但如果每次赋值都执行深拷贝，当数据量巨大时性能将难以承受。例如，一个包含 10,000 个元素的数组：

```swift
var array1 = [1, 2, 3, /* ... 10000 elements */]
var array2 = array1  // 难道要拷贝全部元素吗？
```

Swift 的解决方案是**写时复制**（Copy-on-Write，简称 COW）。简单来说：**拷贝时只共享引用，修改时才真正复制**。

```swift
var numbers = [1, 2, 3, 4, 5]
var copy = numbers      // 此时未复制底层存储，两者共享同一缓冲区

copy[0] = 99            // 修改时触发复制，numbers 不受影响
print(numbers[0])       // 1
```

标准库中的 `Array`、`Dictionary`、`Set` 和 `String` 都实现了 COW 机制。我们也可以在自己定义的结构体中实现 COW。

```swift
final class Storage<T> {
    var items: [T]
    init(_ items: [T]) { self.items = items }
}

struct Stack<T> {
    private var storage: Storage<T>
    
    init(_ items: [T]) {
        self.storage = Storage(items)
    }
    
    // 确保唯一引用的辅助方法
    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = Storage(storage.items)
        }
    }
    
    mutating func push(_ item: T) {
        ensureUnique()
        storage.items.append(item)
    }
    
    mutating func pop() -> T? {
        ensureUnique()
        return storage.items.popLast()
    }
    
    var count: Int {
        storage.items.count
    }
}

var stack1 = Stack([1, 2, 3])
var stack2 = stack1  // 共享 Storage

stack2.push(4)       // 触发 COW，stack1 不受影响
print(stack1.count)  // 3
print(stack2.count)  // 4
```

`isKnownUniquelyReferenced` 是 Swift 运行时提供的关键函数，它检查一个引用类型对象是否只有一个强引用。如果是，我们就可以直接修改而无需复制，从而大幅提升性能。

## 7.4 枚举的威力：关联值与模式匹配

Swift 的枚举远超 C 语言的简单整数枚举，它支持**关联值**（Associated Values），允许每个 case 携带自定义数据。

```swift
enum NetworkResponse {
    case success(data: Data, statusCode: Int)
    case failure(error: Error, retryAfter: TimeInterval?)
    case cached(data: Data, timestamp: Date)
}

// 使用枚举
func handleResponse(_ response: NetworkResponse) {
    switch response {
    case .success(let data, let code):
        print("成功：状态码 \(code)，数据 \(data.count) 字节")
    case .failure(let error, let retryAfter):
        print("失败：\(error.localizedDescription)，\(retryAfter ?? 0) 秒后重试")
    case .cached(let data, let timestamp):
        print("缓存数据：从 \(timestamp) 起，\(data.count) 字节")
    }
}
```

**模式匹配**是枚举的绝佳搭档。`switch` 语句支持丰富的模式匹配语法：

```swift
enum Currency {
    case cny(amount: Double)
    case usd(amount: Double)
    case eur(amount: Double)
}

func description(of currency: Currency) -> String {
    switch currency {
    case .cny(let amount) where amount > 10000:
        return "巨额人民币：¥\(amount)"
    case .cny(let amount):
        return "人民币：¥\(amount)"
    case .usd(let amount) where amount > 1000:
        return "大额美元：$\(amount)"
    case .usd(let amount):
        return "美元：$\(amount)"
    case .eur(let amount):
        return "欧元：€\(amount)"
    }
}

print(description(of: .cny(amount: 15000)))  // "巨额人民币：¥15000.0"
```

还可以使用 `if case` 或 `guard case` 进行单分支匹配：

```swift
let response = NetworkResponse.success(data: Data([0x01, 0x02]), statusCode: 200)

if case .success(let data, _) = response, data.count > 0 {
    print("收到非空数据")
}

guard case .cached(_, let timestamp) = response else {
    // 只有非缓存响应才需要网络请求
    return
}
```

## 7.5 枚举与状态机：用枚举建模有限状态

枚举是建模**有限状态机**（Finite State Machine, FSM）的利器。每个状态可以精确描述其允许的数据和转换规则。

考虑一个网络下载任务的状态机：

```swift
enum DownloadState {
    case idle
    case downloading(progress: Double)
    case paused(progress: Double)
    case completed(data: Data)
    case failed(error: Error)
}
```

我们可以进一步约束状态转换的合法性：

```swift
enum DownloadTaskState {
    case idle
    case downloading(progress: Double)
    case paused(progress: Double)
    case completed(data: Data)
    case failed(error: Error)
    
    /// 是否允许开始下载
    var canStartDownload: Bool {
        if case .idle = self { return true }
        if case .failed = self { return true }
        return false
    }
    
    /// 是否允许暂停
    var canPause: Bool {
        if case .downloading = self { return true }
        return false
    }
    
    /// 是否允许继续
    var canResume: Bool {
        if case .paused = self { return true }
        return false
    }
}

struct DownloadTask {
    private(set) var state: DownloadTaskState = .idle
    
    mutating func start() {
        guard state.canStartDownload else {
            print("当前状态不允许开始下载")
            return
        }
        state = .downloading(progress: 0.0)
    }
    
    mutating func updateProgress(_ progress: Double) {
        guard case .downloading = state else { return }
        if progress >= 1.0 {
            state = .completed(data: Data())
        } else {
            state = .downloading(progress: progress)
        }
    }
    
    mutating func pause() {
        guard case .downloading(let progress) = state else { return }
        state = .paused(progress: progress)
    }
    
    mutating func resume() {
        guard case .paused(let progress) = state else { return }
        state = .downloading(progress: progress)
    }
    
    mutating func fail(with error: Error) {
        guard case .downloading = state else { return }
        state = .failed(error: error)
    }
}
```

这种设计的价值在于**编译器强制检查**。如果你想在不允许的状态下执行非法操作，代码甚至无法通过编译，或者会在运行时被安全拦截。相比于用整数或字符串表示状态，枚举提供了类型安全的保证。

## 7.6 可选就是枚举的应用：Optional\<Wrapped\> 的实现

Swift 中无处不在的可选类型（`Optional`）本质上就是一个枚举。标准库中它的核心定义如下：

```swift
// Swift 标准库中的 Optional 定义（简化）
public enum Optional<Wrapped> {
    case none        // 表示没有值
    case some(Wrapped)  // 包装一个值
}
```

这意味着 `Int?` 只是 `Optional<Int>` 的语法糖。我们可以用枚举语法显式地创建和操作可选值：

```swift
let explicitNone: Optional<Int> = .none
let explicitSome: Optional<Int> = .some(42)

// 与下面的语法糖等价
let implicitNone: Int? = nil
let implicitSome: Int? = 42

// switch 匹配可选值
func describe(_ value: Int?) -> String {
    switch value {
    case .none:
        return "没有值"
    case .some(let x) where x < 0:
        return "负值：\(x)"
    case .some(let x):
        return "值：\(x)"
    }
}

print(describe(42))    // "值：42"
print(describe(nil))   // "没有值"
```

理解 `Optional` 是枚举，能帮助我们更好地理解 Swift 的**可选链**（Optional Chaining）和**可选绑定**（Optional Binding）等语法糖背后的原理：

```swift
// 可选绑定 if-let 的本质
if let value = optionalValue {
    print(value)
}

// 等价于
switch optionalValue {
case .some(let value):
    print(value)
case .none:
    break
}
```

不仅如此，自定义类似 Optional 的泛型枚举，在实际开发中也非常有用：

```swift
/// 表示"加载中"的状态
enum LoadingState<Value> {
    case idle
    case loading
    case loaded(Value)
    case failed(Error)
    
    var value: Value? {
        if case .loaded(let v) = self { return v }
        return nil
    }
    
    var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }
    
    var error: Error? {
        if case .failed(let e) = self { return e }
        return nil
    }
}

// 在 SwiftUI 中管理异步加载数据
class UserViewModel {
    var users: LoadingState<[String]> = .idle
    
    func loadUsers() {
        users = .loading
        // 模拟网络请求
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) { [weak self] in
            self?.users = .loaded(["Alice", "Bob", "Charlie"])
        }
    }
}
```

掌握结构体和枚举，意味着你真正理解了 Swift 区别于其他语言的核心设计哲学——**优先选择值类型**。值类型让代码更加可预测，减少了隐式共享带来的 bug，同时通过 COW 机制兼顾了性能。枚举的关联值和模式匹配能力，则让状态建模和数据表达变得异常优雅和安全。在下一章中，我们将探讨类的正确使用场景，看看什么时候值类型不够用，必须借助引用类型的特性。
