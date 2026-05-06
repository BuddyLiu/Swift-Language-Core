# 第5章 协议扩展与默认实现

## 5.1 扩展的魔力：extension 为已有类型添加功能

Swift 的 `extension` 是一种强大的语言特性，允许我们**为任何已有的类型添加新的功能**，即使没有该类型的源代码。扩展可以添加计算属性、方法、下标、嵌套类型，甚至使类型遵守新的协议。

### 扩展的基础语法

```swift
extension 类型名 {
    // 新的功能
}
```

### 为内置类型添加功能

```swift
// 为 Int 添加一个平方方法
extension Int {
    var squared: Int {
        return self * self
    }
    
    func repeated(operation: () -> Void) {
        for _ in 0..<self {
            operation()
        }
    }
}

print(5.squared)  // 25

3.repeated {
    print("Hello!")  // 打印 3 次
}

// 为 String 添加下标
extension String {
    subscript(index: Int) -> Character {
        return self[self.index(self.startIndex, offsetBy: index)]
    }
    
    var isEmail: Bool {
        return contains("@") && contains(".")
    }
}

print("Hello"[1])       // "e"
print("test@example.com".isEmail)  // true
```

### 使用扩展遵守协议

这是 extension 最有价值的用途之一——让现有类型在不改变定义的情况下遵守协议：

```swift
// 假设有这样一个协议
protocol JSONRepresentable {
    var jsonValue: String { get }
}

// 通过扩展让 Int 遵守协议
extension Int: JSONRepresentable {
    var jsonValue: String {
        return "\(self)"
    }
}

// 让 Array 在元素满足条件时也遵守
extension Array: JSONRepresentable where Element: JSONRepresentable {
    var jsonValue: String {
        let items = self.map { $0.jsonValue }.joined(separator: ", ")
        return "[\(items)]"
    }
}

print(42.jsonValue)           // 42
print([1, 2, 3].jsonValue)   // [1, 2, 3]
```

### 扩展的局限性

```swift
extension SomeType {
    // ✅ 可以添加：计算属性、方法、下标、嵌套类型、协议遵守
    
    // ❌ 不可以添加：存储属性（包括属性观察器）
    // var storedProperty: Int = 0  // 编译错误
    
    // ❌ 不可以添加：指定初始化器（值类型可以添加便利初始化器）
    // 类不能通过扩展添加指定初始化器或反初始化器
}
```

## 5.2 为协议提供默认方法：protocol extension

协议本身只定义"做什么"，不定义"怎么做"。但结合 extension，我们可以为协议方法提供**默认实现**。

### 基本用法

```swift
protocol Greetable {
    var name: String { get }
    func greet()
}

// 通过协议扩展提供默认实现
extension Greetable {
    func greet() {
        print("Hello, \(name)!")
    }
}

// 不提供自定义实现，使用默认实现
struct Person: Greetable {
    let name: String
}

Person(name: "Alice").greet()  // Hello, Alice!

// 也可以覆盖默认实现
struct Robot: Greetable {
    let name: String
    
    func greet() {
        print("Beep boop. Greetings, human \(name).")
    }
}

Robot(name: "Bob").greet()  // Beep boop. Greetings, human Bob.
```

### 默认实现的实际价值

协议扩展让"面向协议编程"真正变得实用。想象没有默认实现时的痛苦：

```swift
// 没有默认实现：每个遵守者都要重复写
protocol Drawable {
    func draw()
    func render()
    func setup()
    func cleanup()
}

// 有默认实现：只需实现关键方法
extension Drawable {
    func render() {
        setup()
        draw()
        cleanup()
    }
    
    func setup() {}     // 空实现，可选覆盖
    func cleanup() {}   // 空实现，可选覆盖
}

struct Triangle: Drawable {
    func draw() {
        print("Drawing triangle")
    }
    // render() 使用默认实现
    // setup() 和 cleanup() 使用空实现
}
```

### 使用扩展提供协议属性的默认实现

```swift
protocol Configurable {
    var timeout: TimeInterval { get }
    var maxRetries: Int { get }
    var baseURL: String { get }
}

// 为所有属性提供默认值
extension Configurable {
    var timeout: TimeInterval { 30.0 }
    var maxRetries: Int { 3 }
    var baseURL: String { "https://api.example.com" }
}

// 只需覆盖需要修改的属性
struct FastConfig: Configurable {
    var timeout: TimeInterval { 5.0 }
    // maxRetries 和 baseURL 使用默认值
}

struct CustomConfig: Configurable {
    var timeout: TimeInterval { 60.0 }
    var maxRetries: Int { 10 }
    var baseURL: String { "https://custom.api.com" }
}
```

## 5.3 动态派发与静态派发的抉择

### 核心概念

- **静态派发（Static Dispatch）**：编译时确定要调用的方法实现。速度快，支持内联优化。
- **动态派发（Dynamic Dispatch）**：运行时通过虚函数表（vtable）或消息机制确定实现。灵活但略有性能开销。

### 协议扩展中的派发规则

理解派发规则是避免 Bug 的关键。Swift 的规则是：

> **如果方法在协议定义中声明，使用动态派发；如果方法仅在协议扩展中定义而未在协议中声明，使用静态派发。**

```swift
protocol Drawing {
    func draw()           // 协议中声明 → 动态派发
}

extension Drawing {
    func draw() {         // 协议扩展中的默认实现
        print("Default draw")
    }
    
    func render() {       // 仅在扩展中有，协议未声明 → 静态派发
        print("Default render")
        draw()            // 这里调用 draw() 是动态派发
    }
}

struct Circle: Drawing {
    func draw() {         // 覆盖默认实现
        print("Circle draw")
    }
    
    func render() {       // 这是新方法，不是覆盖
        print("Circle render")
    }
}

let circle: Drawing = Circle()
circle.draw()     // Circle draw（动态派发）
circle.render()   // Default render（静态派发！）

// 如果类型静态类型是 Circle：
let circle2: Circle = Circle()
circle2.draw()    // Circle draw
circle2.render()  // Circle render（此时调用 Circle 的 render）
```

### 深入理解派发行为

```swift
protocol Vehicle {
    func start()          // 协议中声明 → 动态
    func stop()           // 协议中声明 → 动态
}

extension Vehicle {
    func start() {        // 默认实现
        print("Vehicle started")
    }
    
    func stop() {         // 默认实现
        print("Vehicle stopped")
    }
    
    func honk() {         // 仅扩展中 → 静态
        print("Beep beep!")
    }
}

struct Car: Vehicle {
    func start() {
        print("Car engine started")
    }
    
    func honk() {
        print("Car: Honk honk!")
    }
}

let myCar: Vehicle = Car()
myCar.start()    // Car engine started（动态）
myCar.stop()     // Vehicle stopped（动态，Car 没有覆盖）
myCar.honk()     // Beep beep!（静态！Car 的 honk 不会被调用）

// 关键区别：如果将 myCar 声明为 Car 类型
let anotherCar: Car = Car()
anotherCar.start()  // Car engine started
anotherCar.stop()   // Vehicle stopped
anotherCar.honk()   // Car: Honk honk!
```

> **经验法则**：如果你希望遵守者能够覆盖扩展中的方法，请确保该方法在协议定义中有声明。

## 5.4 协议扩展中的约束条件：where 子句

`where` 子句允许我们为协议扩展添加约束，使得默认实现仅在特定条件下可用。

### 对关联类型的约束

```swift
protocol Container {
    associatedtype Item
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

// 为 Item 是 Equatable 的 Container 提供扩展
extension Container where Item: Equatable {
    func contains(_ item: Item) -> Bool {
        for i in 0..<count {
            if self[i] == item {
                return true
            }
        }
        return false
    }
}

struct StringBox: Container {
    typealias Item = String
    private var items: [String] = []
    
    var count: Int { items.count }
    mutating func append(_ item: String) { items.append(item) }
    subscript(i: Int) -> String { items[i] }
}

var box = StringBox()
box.append("Apple")
box.append("Banana")
print(box.contains("Apple"))  // true（String 是 Equatable）

// 如果 Item 不是 Equatable，contains 方法不可用
struct NotEquatable {}
struct BadBox: Container {
    typealias Item = NotEquatable
    private var items: [NotEquatable] = []
    var count: Int { items.count }
    mutating func append(_ item: NotEquatable) { items.append(item) }
    subscript(i: Int) -> NotEquatable { items[i] }
}
// box.contains(...)  // 编译错误！
```

### 对遵守者自身的约束

```swift
protocol DataSource {
    associatedtype Data
    var data: Data { get }
}

// 仅当 DataSource 本身是 Equatable 时
extension DataSource where Self: Equatable {
    func isIdentical(to other: Self) -> Bool {
        return self == other
    }
}

struct IntSource: DataSource, Equatable {
    var data: Int
}

let source1 = IntSource(data: 42)
let source2 = IntSource(data: 42)
print(source1.isIdentical(to: source2))  // true
```

### 约束指定协议

```swift
protocol Drawable {
    func draw()
}

protocol Transformable {
    func transform()
}

// 扩展：当类型同时遵守 Drawable 和 Transformable 时
extension Drawable where Self: Transformable {
    func drawAndTransform() {
        draw()
        transform()
    }
}

struct Shape: Drawable, Transformable {
    func draw() { print("Drawing") }
    func transform() { print("Transforming") }
}

Shape().drawAndTransform()  // Drawing / Transforming
```

### 实战例子：集合处理

```swift
extension Collection where Element: Numeric {
    func sum() -> Element {
        return reduce(0, +)
    }
}

extension Collection where Element: Comparable {
    func median() -> Element? {
        guard !isEmpty else { return nil }
        let sorted = self.sorted()
        let mid = count / 2
        return sorted[mid]
    }
}

let numbers = [3, 1, 4, 1, 5]
print(numbers.sum())    // 14
print(numbers.median()) // 4

// 字符串数组没有 sum() 方法
let words = ["Hello", "World"]
// words.sum()  // 编译错误！String 不是 Numeric
```

## 5.5 小心陷阱：同名方法的分派规则

当协议扩展提供默认实现，而遵守者也提供了同名方法时，分派规则可能产生令人意外的结果。这是 Swift 开发者最常遇到的陷阱之一。

### 陷阱一：协议方法 vs 扩展方法

```swift
protocol P {
    func method1()  // 声明在协议中
}

extension P {
    func method1() { print("P: method1") }
    func method2() { print("P: method2") }
}

struct S: P {
    func method1() { print("S: method1") }
    func method2() { print("S: method2") }
}

let a: P = S()
let b: S = S()

a.method1()  // S: method1 ✅ 动态派发
a.method2()  // P: method2 ❗ 静态派发！不是 S: method2
b.method1()  // S: method1
b.method2()  // S: method2
```

### 陷阱二：继承链中的分派

```swift
protocol Base {
    func baseMethod()           // 协议声明
    func extendedMethod()       // 协议声明
}

extension Base {
    func baseMethod() { print("Base: baseMethod") }
    func extendedMethod() { print("Base: extendedMethod") }
    func extraMethod() { print("Base: extraMethod") }   // 仅扩展
}

protocol Derived: Base {
    // 继承 Base 的所有要求
}

extension Derived {
    func extendedMethod() { print("Derived: extendedMethod") }
    // 提供 extendedMethod 的另一个默认实现
}

struct MyStruct: Derived {
    func baseMethod() { print("MyStruct: baseMethod") }
}

let instance: Derived = MyStruct()
instance.baseMethod()       // MyStruct: baseMethod（动态）
instance.extendedMethod()   // Derived: extendedMethod（动态）
instance.extraMethod()      // Base: extraMethod（静态）

// 如果声明为 Base 类型
let baseInstance: Base = MyStruct()
baseInstance.extendedMethod()  // Base: extendedMethod ❗ 意外吧？
// 因为 Base.extendedMethod 也有默认实现，而且 Base 的扩展没意识到 Derived 的存在
```

### 如何避免陷阱

**原则1：始终将公共接口声明在协议中**

```swift
// ✅ 好的做法：所有可覆盖的方法都在协议中声明
protocol Good {
    func mustOverride()
    func canOverride()
}

extension Good {
    func mustOverride() { /* 默认实现 */ }
    func canOverride() { /* 默认实现，允许覆盖 */ }
    func cannotOverride() { /* 这个不能覆盖，仅扩展方法 */ }
}

// ❌ 不好的做法：将"可覆盖"的方法只放在扩展中
protocol Bad {
    // 空的协议定义
}

extension Bad {
    func method() { /* 如果我想让遵守者覆盖这个呢？抱歉做不到 */ }
}
```

**原则2：不要依赖协议扩展方法的动态派发**

```swift
protocol Shape {
    var area: Double { get }
}

extension Shape {
    // 提供 description，但不声明在协议中
    // 遵守者如果想覆盖，必须显式声明类型为 Shape？或者使用自己的类型
    var description: String {
        return "Shape with area \(area)"
    }
}

struct Circle: Shape {
    var radius: Double
    var area: Double { .pi * radius * radius }
    
    var description: String {
        return "Circle(radius: \(radius))"
    }
}

let shape: Shape = Circle(radius: 5)
print(shape.description)  // Shape with area 78.539...
// 因为 description 不是协议要求，静态分派到 Shape 的实现
```

**原则3：使用协议扩展提供工具方法而非核心接口**

```swift
protocol Loggable {
    var logTag: String { get }
    func log(_ message: String)
}

extension Loggable {
    // 工具方法：基于 log 方法构建
    func logError(_ error: Error) {
        log("[ERROR] \(error.localizedDescription)")
    }
    
    func logInfo(_ message: String) {
        log("[INFO] \(message)")
    }
    
    func logDebug(_ message: String) {
        #if DEBUG
        log("[DEBUG] \(message)")
        #endif
    }
}

// 遵守者只需要实现核心方法
struct NetworkManager: Loggable {
    var logTag: String { "Network" }
    
    func log(_ message: String) {
        print("[\(logTag)] \(message)")
    }
    // logError、logInfo、logDebug 免费获得
}
```

## 本章小结

协议扩展是 Swift 面向协议编程的"完成形态"，它将协议从"接口定义"升华为"行为库"：

1. **extension**：为任何已有类型添加新功能，包括遵守新协议
2. **协议扩展**：为协议方法提供默认实现，大幅减少重复代码
3. **派发规则**：协议中声明的方法使用动态派发，仅在扩展中定义的方法使用静态派发
4. **where 约束**：让协议扩展在特定条件下才可用，保持类型安全
5. **避免陷阱**：理解同名方法的分派规则，将核心接口声明在协议定义中

掌握协议扩展，是通向真正面向协议编程的必经之路。下一章我们将探讨**泛型**——它与协议结合，构成了 Swift 类型系统最强大的两个支柱。
