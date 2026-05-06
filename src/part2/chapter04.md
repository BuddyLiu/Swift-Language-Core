# 第4章 协议：描述能力而非身份

## 4.1 从面向对象到面向协议：POP vs OOP

在传统的面向对象编程（OOP）中，我们习惯通过继承来复用代码。"猫是动物，所以猫继承自动物"——这种基于身份（Identity）的建模方式长期以来一直是主流。然而，Swift 提供了一种更灵活、更强大的范式：面向协议编程（Protocol-Oriented Programming，POP）。

### OOP 的困境

传统的类继承存在几个固有问题：

1. **脆弱的基类问题**：修改基类可能意外破坏子类的行为
2. **菱形继承**：多继承语言中容易产生歧义
3. **继承强加的身份**：子类"是一个"父类，这种关系有时并不合理

```swift
// OOP 的典型继承结构
class Animal {
    func makeSound() { print("...") }
    func eat() { print("Eating...") }
}

class Dog: Animal {
    override func makeSound() { print("Woof!") }
}

class Cat: Animal {
    override func makeSound() { print("Meow!") }
}

// 如果想让 Bird 既能飞又能叫？
// 多重继承？Swift 不支持
// 那就只能往 Animal 里加 fly() 方法，但狗不会飞
// 或者再建一个层级 FlyingAnimal -> Animal —— 继承层次越来越深
```

### POP 的思维转变

面向协议编程的核心思想是：**关注类型能做什么，而不是它是什么**。协议不关心对象的身份，只描述能力。

```swift
// POP：用协议描述能力
protocol SoundMakable {
    func makeSound()
}

protocol Flyable {
    func fly()
}

struct Dog: SoundMakable {
    func makeSound() { print("Woof!") }
}

struct Bird: SoundMakable, Flyable {
    func makeSound() { print("Chirp!") }
    func fly() { print("Flying...") }
}

// 如今值类型（struct/enum）也可以拥有多态能力
// 不需要继承，只需要遵守协议
```

### POP 的核心优势

| 维度 | OOP | POP |
|------|-----|-----|
| 关注点 | 类型是什么（身份） | 类型能做什么（能力） |
| 代码复用 | 通过继承 | 通过协议扩展 |
| 类型支持 | 仅类（class） | 所有类型（struct/enum/class） |
| 耦合度 | 高（父子类绑定） | 低（协议独立于类型） |
| 多继承 | 不支持或复杂 | 协议天然支持组合 |

## 4.2 协议语法与遵循

### 定义协议

协议定义了一组属性和方法的声明，但不提供实现：

```swift
protocol Drawable {
    // 可读写的实例属性
    var lineWidth: Double { get set }
    
    // 只读的实例属性
    var area: Double { get }
    
    // 类型属性
    static var defaultColor: String { get }
    
    // 实例方法
    func draw()
    
    // 变异方法（mutating 允许值类型修改自身）
    mutating func reset()
    
    // 下标
    subscript(index: Int) -> Double { get }
}
```

### 类型遵守协议

```swift
struct Circle: Drawable {
    var lineWidth: Double = 1.0
    var radius: Double
    
    // 计算属性满足协议要求
    var area: Double {
        return .pi * radius * radius
    }
    
    static var defaultColor: String = "Black"
    
    func draw() {
        print("Drawing a circle with radius \(radius)")
    }
    
    mutating func reset() {
        lineWidth = 1.0
    }
    
    subscript(index: Int) -> Double {
        return radius * Double(index + 1)
    }
}

// 一个类型可以遵守多个协议
struct Rectangle: Drawable, CustomStringConvertible {
    var width: Double
    var height: Double
    var lineWidth: Double = 2.0
    
    var area: Double { width * height }
    static var defaultColor: String = "Blue"
    
    func draw() {
        print("Drawing \(width)x\(height) rectangle")
    }
    
    mutating func reset() {
        lineWidth = 2.0
    }
    
    subscript(index: Int) -> Double {
        return index == 0 ? width : height
    }
    
    // CustomStringConvertible 要求
    var description: String {
        return "Rectangle(\(width) x \(height))"
    }
}
```

### 协议中的属性要求

协议中声明的属性必须明确标注 `{ get }` 或 `{ get set }`：

```swift
protocol Vehicle {
    var speed: Double { get set }   // 可读可写
    var maxSpeed: Double { get }    // 至少可读
}

// 用存储属性满足 { get set }
struct Car: Vehicle {
    var speed: Double      // 存储属性，可读可写
    let maxSpeed: Double   // 存储属性，满足 { get }
}

// 用计算属性满足 { get set }
class Bicycle: Vehicle {
    private var _speed: Double = 0
    
    var speed: Double {
        get { _speed }
        set { _speed = min(newValue, maxSpeed) }
    }
    
    let maxSpeed: Double = 40.0
}
```

### 协议的可选要求

使用 `@objc` 标记协议，可以声明可选要求（仅适用于类）：

```swift
@objc protocol DataSource {
    @objc optional func numberOfItems() -> Int
    @objc optional func item(at index: Int) -> Any
}

class MyDataSource: NSObject, DataSource {
    // 可以不实现可选方法
    func numberOfItems() -> Int { return 10 }
}
```

> **注意**：`@objc` 协议只能被类遵守，且需要 Objective-C 运行时支持。在纯 Swift 场景中，更推荐使用协议扩展来提供默认实现（见第5章）。

## 4.3 协议作为类型使用

协议不仅仅是一种约束，它本身也是一种**类型**，可以作为函数参数、返回值、变量类型来使用。

### 协议作为参数类型

```swift
protocol Drivable {
    func drive()
}

struct Car: Drivable {
    func drive() { print("Car is driving") }
}

struct Truck: Drivable {
    func drive() { print("Truck is driving") }
}

// 接受任意遵守 Drivable 的类型
func testDrive(_ vehicle: Drivable) {
    vehicle.drive()
}

let car = Car()
let truck = Truck()
testDrive(car)    // Car is driving
testDrive(truck)  // Truck is driving
```

### 协议作为返回值类型

```swift
protocol Shape {
    var area: Double { get }
}

struct Square: Shape {
    let side: Double
    var area: Double { side * side }
}

struct Circle: Shape {
    let radius: Double
    var area: Double { .pi * radius * radius }
}

// 工厂函数，返回协议类型
func randomShape() -> Shape {
    return Bool.random() 
        ? Square(side: 4)
        : Circle(radius: 3)
}

let shape = randomShape()
print("Area: \(shape.area)")
```

### 协议类型的集合

```swift
let shapes: [Shape] = [
    Square(side: 2),
    Circle(radius: 5),
    Square(side: 3)
]

let totalArea = shapes.reduce(0) { $0 + $1.area }
print("Total area: \(totalArea)")
```

### 协议类型的协议

可以通过 `is` 和 `as?` 进行类型检查和向下转型：

```swift
for shape in shapes {
    if let square = shape as? Square {
        print("Square with side \(square.side)")
    } else if shape is Circle {
        print("It's a circle!")
    }
}
```

## 4.4 协议继承与组合

### 协议继承

协议可以继承其他协议，在已有要求之上添加新的要求：

```swift
protocol Machine {
    var power: Double { get set }
    func start()
    func stop()
}

// Printer 继承 Machine，添加了打印相关的要求
protocol Printer: Machine {
    var paperCount: Int { get set }
    func print(document: String)
}

// Scanner 继承 Machine，添加了扫描相关的要求
protocol Scanner: Machine {
    var resolution: Int { get }
    func scan() -> Data
}

// 多协议继承
protocol AllInOne: Printer, Scanner {
    func fax()
}
```

### 遵守继承的协议

```swift
struct OfficePrinter: Printer {
    var power: Double = 100
    var paperCount: Int = 500
    
    func start() {
        print("Printer ready")
    }
    
    func stop() {
        print("Printer shutting down")
    }
    
    func print(document: String) {
        guard paperCount > 0 else {
            print("No paper!")
            return
        }
        print("Printing: \(document)")
        paperCount -= 1
    }
}
```

### 协议组合（Protocol Composition）

Swift 使用 `&` 符号将多个协议组合成一个临时的复合类型：

```swift
protocol Loggable {
    var log: String { get }
}

protocol Encodable {
    func encode() -> Data
}

// 组合：类型必须同时遵守 Loggable 和 Encodable
func processItem(_ item: Loggable & Encodable) {
    print("Log: \(item.log)")
    let data = item.encode()
    // 处理 data...
}

// 还可以与类类型组合
func configureUI(_ view: UIView & Loggable) {
    view.addSubview(/* ... */)
    print("Configured: \(view.log)")
}
```

### 组合的实际应用

协议组合在 SwiftUI 和日常开发中非常常见：

```swift
// 一个更真实的例子：网络请求的响应处理
protocol Responsable {
    associatedtype Response
    var statusCode: Int { get }
    var data: Response { get }
}

protocol ErrorHandleable {
    var error: Error? { get }
    func handleError() -> String
}

protocol Cacheable {
    var cacheKey: String { get }
    func cachedResponse() -> Data?
}

// 组合使用
typealias RobustResponse = Responsable & ErrorHandleable & Cacheable

func handleResponse<T: RobustResponse>(_ response: T) -> String {
    if let error = response.error {
        return error.localizedDescription
    }
    
    if let cached = response.cachedResponse() {
        return "Loaded from cache: \(cached.count) bytes"
    }
    
    return "Processing response (\(response.statusCode))..."
}
```

## 4.5 Self 与关联类型

### Self 关键字

在协议中，`Self` 指向最终遵守该协议的具体类型：

```swift
protocol Cloneable {
    // 返回遵守该协议的具体类型自身
    func clone() -> Self
}

class Account: Cloneable {
    var balance: Double = 0
    
    func clone() -> Self {
        // 注意：这里必须用 Self，不能写 Account
        let newAccount = type(of: self).init()
        newAccount.balance = self.balance
        return newAccount
    }
    
    required init() {}  // 需要 required init 支持
}
```

### 关联类型 associatedtype

关联类型为协议提供了一种方式，允许遵循者在实现时指定具体的类型：

```swift
protocol Container {
    // 关联类型：由遵守者在实现时指定
    associatedtype Item
    
    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}
```

### 实现关联类型协议

```swift
// 泛型类型实现
struct Stack<T>: Container {
    // typealias Item = T  // 可显式指定，也可由编译器推断
    private var items: [T] = []
    
    var count: Int { items.count }
    
    mutating func append(_ item: T) {
        items.append(item)
    }
    
    subscript(i: Int) -> T {
        return items[i]
    }
    
    mutating func pop() -> T? {
        return items.popLast()
    }
}

// 非泛型类型实现：具体指定关联类型
struct IntList: Container {
    typealias Item = Int  // 明确指定
    private var items: [Int] = []
    
    var count: Int { items.count }
    
    mutating func append(_ item: Int) {
        items.append(item)
    }
    
    subscript(i: Int) -> Int {
        return items[i]
    }
}
```

### 关联类型的约束

可以对关联类型添加约束：

```swift
protocol ComparableContainer {
    associatedtype Item: Comparable  // Item 必须遵守 Comparable
    
    var items: [Item] { get }
    func sorted() -> [Item]
}

extension ComparableContainer {
    func sorted() -> [Item] {
        return items.sorted()
    }
}

struct NumberBox: ComparableContainer {
    typealias Item = Int  // Int 遵守 Comparable
    var items: [Int]
}
```

### 使用 where 约束关联类型

```swift
protocol Sequence {
    associatedtype Element
    associatedtype Iterator: IteratorProtocol where Iterator.Element == Element
    
    func makeIterator() -> Iterator
}
```

## 4.6 标准库中的经典协议

### Equatable（可判等）

```swift
struct Point: Equatable {
    var x: Double
    var y: Double
    
    // 编译器可自动合成 == 的实现
    // 无需手动编写
}

// 自动合成的前提：所有存储属性都是 Equatable
let p1 = Point(x: 1, y: 2)
let p2 = Point(x: 1, y: 2)
print(p1 == p2)  // true
print(p1 != p2)  // false

// 也可以自定义判等逻辑
struct User: Equatable {
    var id: Int
    var name: String
    var email: String
    
    // 只根据 id 判等
    static func == (lhs: User, rhs: User) -> Bool {
        return lhs.id == rhs.id
    }
}
```

### Comparable（可比较）

```swift
struct Fraction: Comparable {
    var numerator: Int
    var denominator: Int
    
    static func < (lhs: Fraction, rhs: Fraction) -> Bool {
        // 交叉相乘比较
        return lhs.numerator * rhs.denominator < rhs.numerator * lhs.denominator
    }
    
    // == 也需要 Equatable
    static func == (lhs: Fraction, rhs: Fraction) -> Bool {
        return lhs.numerator * rhs.denominator == rhs.numerator * lhs.denominator
    }
}

let f1 = Fraction(numerator: 1, denominator: 2)
let f2 = Fraction(numerator: 2, denominator: 3)
print(f1 < f2)  // true (0.5 < 0.666...)

// 遵守 Comparable 后自动获得以下能力：
let fractions = [f1, f2].sorted()       // 排序
let range = f1..<f2                     // 可以用在半开区间中
print(f1 >= f2)                         // <=, >, >= 自动生成
```

### Hashable（可哈希）

```swift
struct Product: Hashable {
    var sku: String   // 自动合成
    var name: String  // 自动合成
    
    // Hashable 继承自 Equatable，需要 ==
    // 编译器可以自动合成 hash(into:)
}

let productSet: Set<Product> = [
    Product(sku: "A001", name: "iPhone"),
    Product(sku: "A002", name: "iPad")
]

let productDict: [Product: Int] = [
    Product(sku: "A001", name: "iPhone"): 999
]

// 自定义哈希
struct CustomHash: Hashable {
    var id: Int
    var timestamp: Date
    
    func hash(into hasher: inout Hasher) {
        // 只使用 id 进行哈希
        hasher.combine(id)
    }
    
    static func == (lhs: CustomHash, rhs: CustomHash) -> Bool {
        return lhs.id == rhs.id
    }
}
```

### Codable（可编解码）

Codable 是 `Encodable` 和 `Decodable` 的类型别名，是 Swift 最强大的协议之一：

```swift
struct Person: Codable {
    var name: String
    var age: Int
    var email: String
    
    // 自定义编码键名
    enum CodingKeys: String, CodingKey {
        case name
        case age
        case email = "email_address"  // JSON 中的键名
    }
}

// 编码
let person = Person(name: "Alice", age: 30, email: "alice@example.com")
let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted

do {
    let data = try encoder.encode(person)
    let jsonString = String(data: data, encoding: .utf8)!
    print(jsonString)
} catch {
    print("Encoding failed: \(error)")
}

// 解码
let json = """
{
    "name": "Bob",
    "age": 25,
    "email_address": "bob@example.com"
}
""".data(using: .utf8)!

do {
    let decoded = try JSONDecoder().decode(Person.self, from: json)
    print("Decoded: \(decoded.name), \(decoded.age)")
} catch {
    print("Decoding failed: \(error)")
}
```

### Identifiable（可标识）

Identifiable 是 SwiftUI 和现代 Swift 中广泛使用的协议：

```swift
struct TodoItem: Identifiable {
    // 只要有一个名为 id 的属性，编译器自动合成
    let id: UUID = UUID()
    var title: String
    var isCompleted: Bool
}

// 也可以使用其他类型作为 ID
struct Employee: Identifiable {
    var id: String  // 员工工号作为 ID
    var name: String
    var department: String
}

// 在 SwiftUI 中使用 Identifiable
// List(todos) { item in ... }
// ForEach(employees) { emp in ... }
```

### 协议的综合运用

让我们用一个完整的例子展示这些协议如何协同工作：

```swift
struct Book: Codable, Hashable, Identifiable, Comparable {
    let id: UUID
    var title: String
    var author: String
    var publishYear: Int
    var price: Double
    
    // Hashable 自动合成
    // Codable 自动合成
    
    static func < (lhs: Book, rhs: Book) -> Bool {
        if lhs.publishYear != rhs.publishYear {
            return lhs.publishYear < rhs.publishYear
        }
        if lhs.author != rhs.author {
            return lhs.author < rhs.author
        }
        return lhs.title < rhs.title
    }
}

// 使用示例
var library: Set<Book> = []
let book1 = Book(
    id: UUID(),
    title: "Swift Programming",
    author: "Apple",
    publishYear: 2023,
    price: 39.99
)
library.insert(book1)

// Comparable 允许排序
let sortedBooks = library.sorted()
```

## 本章小结

协议是 Swift 面向协议编程的基石。通过本章的学习，我们理解了：

1. **POP 与 OOP 的区别**：协议关注能力而非身份，值类型也能获得多态能力
2. **协议的语法**：使用 `protocol` 声明，类型通过冒号语法遵守协议
3. **协议作为类型**：可以作为参数、返回值、集合元素类型
4. **协议继承与组合**：协议可以继承其他协议，使用 `&` 进行协议组合
5. **Self 与关联类型**：`Self` 指向具体类型，`associatedtype` 让协议更通用
6. **标准库经典协议**：Equatable、Comparable、Hashable、Codable、Identifiable 是日常开发的核心工具

协议本身只声明要求，不提供实现。那么如何为协议提供默认行为呢？这正是下一章要探讨的**协议扩展**。
