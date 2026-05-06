# 第6章 泛型：复用一切可复用的代码

## 6.1 泛型函数与类型参数

泛型（Generics）是 Swift 最强大的特性之一。它让我们能够编写**不依赖于特定类型**的灵活代码，同时保持类型安全。

### 为什么需要泛型？

先看一个非泛型的例子：

```swift
// 交换两个 Int 值
func swapInts(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

// 交换两个 Double 值
func swapDoubles(_ a: inout Double, _ b: inout Double) {
    let temp = a
    a = b
    b = temp
}

// 交换两个 String 值
func swapStrings(_ a: inout String, _ b: inout String) {
    let temp = a
    a = b
    b = temp
}
```

上述代码存在大量重复，唯一的区别是类型不同。用泛型改写：

```swift
// 泛型函数：一个版本搞定所有类型
func swap<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var x = 10, y = 20
swap(&x, &y)
print("x = \(x), y = \(y)")  // x = 20, y = 10

var hello = "Hello", world = "World"
swap(&hello, &world)
print(hello, world)  // World Hello
```

### 泛型函数语法

```swift
// 基本语法
func 函数名<类型参数列表>(参数列表) -> 返回值 {
    // 函数体
}

// 多个类型参数
func makePair<T, U>(_ first: T, _ second: U) -> (T, U) {
    return (first, second)
}

let pair = makePair("Age", 30)
print(pair)  // ("Age", 30)

// 类型参数名惯例
// T, U, V: 单个类型参数
// T: Element, Key, Value: 有含义的名称
// 泛型容器：Element、Index 等
```

### 类型推断与显式标注

```swift
// 编译器通常可以推断类型参数
let result = makePair(42, "Swift")  // 推断为 (Int, String)

// 也可以显式标注
let explicit: (Double, Bool) = makePair(3.14, true)

// 当编译器无法推断时，需要显式指定
func identity<T>(_ value: T) -> T {
    return value
}

let num = identity(42)         // 推断：Int
let str: String = identity("Hi")  // 通过返回值推断

// 如果上下文不明确，需要显式指定
// let ambiguous = identity  // 错误：泛型类型无法推断
let explicit2: (Int) -> Int = identity  // 明确
```

## 6.2 泛型约束：让协议参与抽象

泛型虽然灵活，但有时我们需要对类型参数施加限制。这就是**泛型约束**的用武之地。

### 基本约束语法

```swift
// 约束 T 必须遵守 Equatable
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    for (index, item) in array.enumerated() {
        if item == value {  // 因为 T: Equatable，所以可以用 ==
            return index
        }
    }
    return nil
}

print(findIndex(of: 42, in: [1, 2, 42, 3]))       // Optional(2)
print(findIndex(of: "Swift", in: ["Java", "Kotlin", "Swift"]))  // Optional(2)

// 如果没有 Equatable 约束，item == value 会编译错误
```

### 多协议约束

```swift
// 约束 T 同时遵守 Comparable 和 CustomStringConvertible
func describeMax<T: Comparable & CustomStringConvertible>(_ a: T, _ b: T) -> String {
    return a > b ? a.description : b.description
}

print(describeMax(42, 100))       // 100
print(describeMax(3.14, 2.71))    // 3.14
```

### 类约束

```swift
// 约束 T 必须是 UIView 或其子类
func configureView<T: UIView>(_ view: T, withColor color: UIColor) -> T {
    view.backgroundColor = color
    view.layer.cornerRadius = 8
    return view
}

// 约束：必须是某个类的子类，同时遵守协议
func animateView<T: UIView & Animatable>(_ view: T) {
    // 同时拥有 UIView 的属性和 Animatable 的方法
    view.animate()
}
```

### where 子句约束

约束也可以使用 `where` 子句，功能更强大：

```swift
// where 子句可以表达更复杂的约束
func allEqual<T>(_ array: [T]) -> Bool where T: Equatable {
    guard let first = array.first else { return true }
    return array.allSatisfy { $0 == first }
}

// 上述写法等价于：
func allEqual2<T: Equatable>(_ array: [T]) -> Bool {
    guard let first = array.first else { return true }
    return array.allSatisfy { $0 == first }
}
```

## 6.3 where 子句深入

`where` 是 Swift 泛型中最灵活的工具，能表达复杂的类型关系。

### 关联类型约束

```swift
protocol Container {
    associatedtype Item
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}

// 约束两个容器的 Item 必须相同
func areEqual<C1: Container, C2: Container>(_ lhs: C1, _ rhs: C2) -> Bool
    where C1.Item: Equatable, C1.Item == C2.Item {
    
    guard lhs.count == rhs.count else { return false }
    for i in 0..<lhs.count {
        if lhs[i] != rhs[i] { return false }
    }
    return true
}
```

### 多层约束

```swift
func sortedMerge<C1: Collection, C2: Collection>(_ c1: C1, _ c2: C2) -> [C1.Element]
    where C1.Element: Comparable,
          C1.Element == C2.Element,
          C1: ExpressibleByArrayLiteral {
    
    let sorted1 = c1.sorted()
    let sorted2 = c2.sorted()
    
    var result: [C1.Element] = []
    var i = 0, j = 0
    
    while i < sorted1.count && j < sorted2.count {
        if sorted1[i] < sorted2[j] {
            result.append(sorted1[i])
            i += 1
        } else {
            result.append(sorted2[j])
            j += 1
        }
    }
    
    while i < sorted1.count { result.append(sorted1[i]); i += 1 }
    while j < sorted2.count { result.append(sorted2[j]); j += 1 }
    
    return result
}

let a = [1, 3, 5]
let b = [2, 4, 6]
print(sortedMerge(a, b))  // [1, 2, 3, 4, 5, 6]
```

### where 与泛型扩展

`where` 在泛型扩展中尤为有用（见下节 6.4 的详细讨论）。

### 协议中的 where 子句

```swift
protocol SelfEquatable where Self: Equatable {
    func isEqualToSelf(_ other: Self) -> Bool
}

extension SelfEquatable {
    func isEqualToSelf(_ other: Self) -> Bool {
        return self == other
    }
}

// 只有 Equatable 的类型才能遵守 SelfEquatable
struct Value: SelfEquatable, Equatable {
    var id: Int
}

// 使用 associatedtype 时的 where
protocol Sequence {
    associatedtype Element
    associatedtype Iterator: IteratorProtocol where Iterator.Element == Element
    func makeIterator() -> Iterator
}
```

## 6.4 泛型类型与泛型扩展

### 泛型类型

Swift 允许定义泛型类型——结构体、类、枚举都可以是泛型的：

```swift
// 泛型栈
struct Stack<Element> {
    private var items: [Element] = []
    
    mutating func push(_ item: Element) {
        items.append(item)
    }
    
    @discardableResult
    mutating func pop() -> Element? {
        return items.popLast()
    }
    
    func peek() -> Element? {
        return items.last
    }
    
    var isEmpty: Bool { items.isEmpty }
    var count: Int { items.count }
}

// 使用
var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
intStack.push(3)
print(intStack.pop()!)  // 3

var stringStack = Stack<String>()
stringStack.push("Hello")
stringStack.push("World")
print(stringStack.peek()!)  // World
```

### 泛型枚举

```swift
// Swift 标准库中的 Optional 本质上是泛型枚举
// enum Optional<Wrapped> {
//     case none
//     case some(Wrapped)
// }

// 自定义泛型枚举
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}

func fetchData() -> Result<String, NetworkError> {
    return .success("Data loaded")
}

enum NetworkError: Error {
    case timeout
    case noConnection
}
```

### 泛型类型的扩展

扩展泛型类型时，类型参数在扩展范围内仍然可用：

```swift
// 基础扩展：所有 Stack 可用
extension Stack {
    var top: Element? {
        return items.last
    }
    
    mutating func pushAll(_ newItems: [Element]) {
        items.append(contentsOf: newItems)
    }
}

// 带 where 子句的扩展：仅 Element 为 Equatable 时可用
extension Stack where Element: Equatable {
    func contains(_ item: Element) -> Bool {
        return items.contains(item)
    }
    
    func removeAll(_ item: Element) {
        items.removeAll { $0 == item }
    }
}

// 更进一步：Element 为特定类型时的扩展
extension Stack where Element == Double {
    var average: Element {
        return items.reduce(0, +) / Double(items.count)
    }
}

var doubleStack = Stack<Double>()
doubleStack.pushAll([1.0, 2.0, 3.0, 4.0])
print(doubleStack.average)  // 2.5

var mixedStack = Stack<String>()
// mixedStack.average  // 编译错误：String 没有 average
```

### 泛型类型的协议遵守

```swift
// 让 Stack 在 Element 为 Codable 时也遵守 Codable
extension Stack: Codable where Element: Codable {
    // Codable 自动合成
}

// 让 Stack 遵守 Collection 协议
extension Stack: Collection {
    var startIndex: Int { 0 }
    var endIndex: Int { count }
    
    func index(after i: Int) -> Int {
        return i + 1
    }
}

// 现在 Stack 可以使用 for-in 循环
var numbers = Stack<Int>()
numbers.pushAll([10, 20, 30])

for item in numbers {
    print(item)  // 10, 20, 30
}

// 以及所有 Collection 的方法
print(numbers.map { $0 * 2 })  // [20, 40, 60]
```

## 6.5 不透明类型 some：向调用者隐藏具体类型

Swift 5.1 引入的 `some` 关键字，让我们可以在**隐藏具体类型**的同时，保持类型关系不变。

### 不透明类型 vs 协议类型

```swift
protocol Shape {
    var area: Double { get }
}

struct Circle: Shape {
    var radius: Double
    var area: Double { .pi * radius * radius }
}

struct Square: Shape {
    var side: Double
    var area: Double { side * side }
}

// 返回协议类型（盒装类型，存在箱 Existential Container）
func makeRandomShape() -> Shape {
    return Bool.random() ? Circle(radius: 5) : Square(side: 4)
}

// 返回不透明类型（保持具体类型信息）
func makeCircle() -> some Shape {
    return Circle(radius: 5)
}

// some 的核心限制：每次返回的类型必须一致
func makeShape() -> some Shape {
    // 编译错误：不同路径返回不同类型
    // return Bool.random() ? Circle(radius: 5) : Square(side: 4)
    return Circle(radius: 5)  // ✅ 始终返回同一类型
}
```

### some 的语义：反向泛型

```swift
// 泛型：调用者决定类型
func create<T: Shape>(_ type: T.Type) -> T {
    return type.init()
}

// some：实现者决定类型
func createDefault() -> some Shape {
    return Circle(radius: 1)
}
```

### some 的实际应用

```swift
// 在 SwiftUI 中大量使用 some
// var body: some View {
//     Text("Hello")
// }

// 避免暴露复杂的嵌套类型
func makeComplexView() -> some View {
    // 实际返回的是：ModifiedContent<Text, _EnvironmentKeyWritingModifier<Optional<Color>>>
    Text("Hello").foregroundColor(.blue)
}

// 集合操作
func makeComputedCollection() -> some Collection {
    return [1, 2, 3].filter { $0 > 1 }.map { $0 * 2 }
    // 实际返回类型是 LazyMapSequence<...>
}

let collection = makeComputedCollection()
print(collection.count)  // 2
// print(collection[0])  // ✅ 不透明类型保留了 Collection 的接口
```

### 不透明类型与泛型的关键区别

```swift
protocol Flyable {
    associatedtype Fuel
    func fly()
}

// ❌ 协议类型不能有 associatedtype
// func makeFlyer() -> Flyable {  // 编译错误

// ✅ 可以用不透明类型
func makeFlyer() -> some Flyable {
    return Bird()
}

struct Bird: Flyable {
    typealias Fuel = Seeds
    func fly() { print("Flap flap!") }
}

struct Seeds {}

// 不透明类型的身份保持
let flyer1 = makeFlyer()
let flyer2 = makeFlyer()
// flyer1 和 flyer2 是同一类型（some Flyable 保证）
```

## 6.6 泛型与协议的协同：面向协议编程的完整形态

泛型和协议是 Swift 类型系统的两翼，它们的结合让面向协议编程展现出真正的威力。

### 泛型协议：associatedtype

```swift
// 协议 + 关联类型 = 泛型协议
protocol Repository {
    associatedtype Entity
    
    func getAll() -> [Entity]
    func get(by id: String) -> Entity?
    func save(_ entity: Entity)
    func delete(_ entity: Entity)
}

// 泛型实现
class InMemoryRepository<T>: Repository {
    typealias Entity = T
    private var storage: [String: T] = [:]
    private var idGenerator: (T) -> String
    
    init(idGenerator: @escaping (T) -> String) {
        self.idGenerator = idGenerator
    }
    
    func getAll() -> [T] {
        Array(storage.values)
    }
    
    func get(by id: String) -> T? {
        storage[id]
    }
    
    func save(_ entity: T) {
        let id = idGenerator(entity)
        storage[id] = entity
    }
    
    func delete(_ entity: T) {
        let id = idGenerator(entity)
        storage.removeValue(forKey: id)
    }
}

// 类型安全的具体仓库
struct User {
    let id: String
    let name: String
}

let userRepo = InMemoryRepository<User> { $0.id }
userRepo.save(User(id: "1", name: "Alice"))
print(userRepo.get(by: "1")!.name)  // Alice
```

### 泛型约束 + 协议扩展

```swift
// 结合泛型约束与协议扩展，实现条件功能
protocol Collection: Sequence {
    associatedtype Element
    var count: Int { get }
}

extension Collection where Element: Comparable {
    func sorted() -> [Element] {
        return Array(self).sorted()
    }
}

extension Collection where Element: Hashable {
    func unique() -> [Element] {
        return Array(Set(self))
    }
}

// 使用
let items = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
print(items.sorted())  // [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]
print(items.unique())  // [3, 1, 4, 5, 9, 2, 6]
```

### 通过泛型构建抽象层

```swift
// 数据源抽象
protocol DataSource {
    associatedtype Item
    func load() -> [Item]
}

protocol DataCache {
    associatedtype Item
    func cache(_ items: [Item])
    func loadCached() -> [Item]?
}

protocol DataTransformer {
    associatedtype Input
    associatedtype Output
    func transform(_ input: [Input]) -> [Output]
}

// 完整的数据管道
class DataPipeline<Source: DataSource, Cache: DataCache, Transformer: DataTransformer>
    where Source.Item == Cache.Item,
          Cache.Item == Transformer.Input {
    
    private let source: Source
    private let cache: Cache
    private let transformer: Transformer
    
    init(source: Source, cache: Cache, transformer: Transformer) {
        self.source = source
        self.cache = cache
        self.transformer = transformer
    }
    
    func process() -> [Transformer.Output] {
        if let cached = cache.loadCached() {
            return transformer.transform(cached)
        }
        
        let data = source.load()
        cache.cache(data)
        return transformer.transform(data)
    }
}

// 具体实现
struct LocalFileSource: DataSource {
    typealias Item = String
    func load() -> [String] {
        return ["apple", "banana", "cherry"]
    }
}

struct MemoryCache: DataCache {
    typealias Item = String
    private var stored: [String]?
    
    func cache(_ items: [String]) { stored = items }
    func loadCached() -> [String]? { stored }
}

struct UppercaseTransformer: DataTransformer {
    typealias Input = String
    typealias Output = String
    
    func transform(_ input: [String]) -> [String] {
        input.map { $0.uppercased() }
    }
}

// 组合使用
let pipeline = DataPipeline(
    source: LocalFileSource(),
    cache: MemoryCache(),
    transformer: UppercaseTransformer()
)

print(pipeline.process())  // ["APPLE", "BANANA", "CHERRY"]
```

### 约束的传递性

```swift
protocol IdentifiableEntity {
    associatedtype ID: Hashable
    var id: ID { get }
}

protocol PersistableEntity: IdentifiableEntity {
    associatedtype Entity: Codable
    func toEntity() -> Entity
}

// 由于 ID: Hashable，Entity 自动获得了 Hashable 约束传递
// 遵守 PersistableEntity 的类型必须同时满足：
// 1. id 的类型是 Hashable
// 2. Entity 的类型是 Codable

struct UserEntity: PersistableEntity {
    typealias ID = UUID        // UUID 是 Hashable
    typealias Entity = UserDTO  // UserDTO 是 Codable
    
    let id: UUID
    var name: String
    var email: String
    
    func toEntity() -> UserDTO {
        UserDTO(id: id, name: name, email: email)
    }
}

struct UserDTO: Codable {
    let id: UUID
    let name: String
    let email: String
}
```

### 泛型协议的最佳实践

```swift
// 1. 使用协议 + 关联类型描述核心能力
protocol Cacheable {
    associatedtype Key: Hashable
    associatedtype Value
    
    func get(_ key: Key) -> Value?
    func set(_ key: Key, value: Value)
    func remove(_ key: Key)
    func clear()
}

// 2. 使用泛型提供默认实现
class BaseCache<K: Hashable, V>: Cacheable {
    typealias Key = K
    typealias Value = V
    
    private var storage: [K: V] = [:]
    
    func get(_ key: K) -> V? { storage[key] }
    func set(_ key: K, value: V) { storage[key] = value }
    func remove(_ key: K) { storage.removeValue(forKey: key) }
    func clear() { storage.removeAll() }
}

// 3. 使用协议扩展提供通用行为
extension Cacheable {
    func getOrDefault(_ key: Key, default: Value) -> Value {
        return get(key) ?? `default`
    }
    
    func exists(_ key: Key) -> Bool {
        return get(key) != nil
    }
}

// 4. 使用 where 子句精细化控制
extension Cacheable where Value: Comparable {
    func highestValue() -> Value? {
        fatalError("Subclass must implement")
    }
}
```

## 本章小结

泛型是 Swift 类型系统的核心支柱之一。本章涵盖了：

1. **泛型函数与类型参数**：用尖括号语法实现类型无关的通用函数
2. **泛型约束**：使用 `: 协议` 语法限制类型参数
3. **where 子句**：表达复杂的类型关系，包括关联类型的约束
4. **泛型类型与扩展**：定义泛型结构体、类、枚举，并通过扩展提供条件功能
5. **不透明类型 some**：隐藏具体类型同时保留类型信息
6. **泛型与协议的协同**：通过 associatedtype、约束和扩展实现完整的面向协议编程

泛型和协议的结合，使 Swift 具备了强大的抽象能力而不牺牲类型安全。这种设计让代码更复用、更安全、更易维护，是 Swift 语言最优雅的设计之一。
