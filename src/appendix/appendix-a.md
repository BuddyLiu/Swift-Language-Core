# 附录A 关键协议与类型的快速参考

Swift 标准库提供了丰富的协议和类型，它们是整个语言生态的基石。本附录以「快速参考卡片」的形式，为每个核心协议和类型提供简要说明、关键方法（或属性）以及示例代码，方便读者在实际开发中速查。

## Equatable

**说明**：允许两个实例进行相等性比较（`==` / `!=`）。Swift 中所有基本类型（`Int`、`String`、`Array` 等）都遵循此协议。结构体如果所有存储属性都遵循 `Equatable`，编译器会自动合成实现。

```swift
struct Point: Equatable {
    var x: Int
    var y: Int
}

let a = Point(x: 1, y: 2)
let b = Point(x: 1, y: 2)
print(a == b) // true
```

## Comparable

**说明**：继承自 `Equatable`，允许使用 `<`、`<=`、`>=`、`>` 进行比较。只需实现 `<` 操作符，其余由协议默认实现提供。

```swift
struct Person: Comparable {
    var name: String
    var age: Int
    
    static func < (lhs: Person, rhs: Person) -> Bool {
        return lhs.age < rhs.age
    }
}

let people = [Person(name: "Alice", age: 30), Person(name: "Bob", age: 25)]
let sorted = people.sorted() // Bob 在前
```

## Hashable

**说明**：使实例能被哈希，从而用于 `Set` 或 `Dictionary` 的键。遵循 `Hashable` 自动意味着也遵循 `Equatable`。编译器通常会为结构体自动合成实现。

```swift
struct Color: Hashable {
    var red: Int
    var green: Int
    var blue: Int
}

let palette: Set<Color> = [
    Color(red: 255, green: 0, blue: 0),
    Color(red: 0, green: 255, blue: 0)
]
```

**关键成员**：
- `func hash(into hasher: inout Hasher)` — 自定义哈希逻辑时实现此方法。

## Encodable & Decodable (Codable)

**说明**：`Encodable` 可将类型实例编码为外部表示（如 JSON）；`Decodable` 则从外部表示解码。`Codable` 是二者的类型别名。大多数字段一一对应的类型只需声明遵循即可。

```swift
struct User: Codable {
    var id: Int
    var name: String
    var email: String
}

let json = """
{"id": 1, "name": "Alice", "email": "alice@example.com"}
""".data(using: .utf8)!

let user = try JSONDecoder().decode(User.self, from: json)
```

**关键类型**：
- `JSONEncoder` / `JSONDecoder`
- `PropertyListEncoder` / `PropertyListDecoder`

## Identifiable

**说明**：要求提供一个稳定且唯一的 `id` 属性，常用于 `ForEach`、`List` 等 SwiftUI 视图中标识元素。

```swift
struct Task: Identifiable {
    var id: UUID = UUID()
    var title: String
    var isCompleted: Bool
}

let tasks = [Task(title: "写代码"), Task(title: "测试")]
// 在 SwiftUI 中：
// List(tasks) { task in Text(task.title) }
```

**关键属性**：
- `var id: ID` — 其中 `ID` 类型需遵循 `Hashable`。

## CustomStringConvertible & CustomDebugStringConvertible

**说明**：允许自定义实例的文本表示。`CustomStringConvertible` 影响 `print()` 和字符串插值；`CustomDebugStringConvertible` 影响 `debugPrint()` 和 lldb 调试器输出。

```swift
struct Vector: CustomStringConvertible {
    var x: Double
    var y: Double
    
    var description: String {
        return "(\(x), \(y))"
    }
}

let v = Vector(x: 3.0, y: 4.0)
print(v) // (3.0, 4.0)
```

**关键属性**：
- `var description: String`（`CustomStringConvertible`）
- `var debugDescription: String`（`CustomDebugStringConvertible`）

## Sequence & Collection

**说明**：`Sequence` 代表可被 `for-in` 遍历的一系列值，只需实现 `makeIterator()`。`Collection` 继承自 `Sequence`，增加下标访问和元素计数，`Array`、`Dictionary`、`Set` 均遵循。

```swift
// Sequence：只需提供迭代器
struct Countdown: Sequence {
    let start: Int
    
    func makeIterator() -> some IteratorProtocol {
        return CountdownIterator(current: start)
    }
}

struct CountdownIterator: IteratorProtocol {
    var current: Int
    
    mutating func next() -> Int? {
        guard current >= 0 else { return nil }
        defer { current -= 1 }
        return current
    }
}

for n in Countdown(start: 5) {
    print(n) // 5, 4, 3, 2, 1, 0
}

// Collection：支持下标
let array: [Int] = [10, 20, 30]
print(array[1]) // 20
print(array.count) // 3
```

**关键成员**：
- `Sequence`: `func makeIterator() -> some IteratorProtocol`
- `Collection`: `var startIndex: Index`、`var endIndex: Index`、`subscript(position: Index) -> Element`、`func index(after i: Index) -> Index`

## IteratorProtocol

**说明**：为 `Sequence` 提供逐个生成元素的能力。只需实现 `next()` 方法，当序列耗尽时返回 `nil`。

```swift
struct Fibonacci: IteratorProtocol {
    var current = 0
    var nextValue = 1
    
    mutating func next() -> Int? {
        let result = current
        current = nextValue
        nextValue = result + nextValue
        // 不返回 nil，生成无限序列
        return result
    }
}

var fib = Fibonacci()
for _ in 0..<10 {
    print(fib.next()!, terminator: " ") // 0 1 1 2 3 5 8 13 21 34
}
```

## OptionSet

**说明**：提供类似 C 语言中位掩码（bitmask）的功能，但类型安全。遵循 `OptionSet` 的类型需要提供一个 `rawValue`（通常为 `Int`），并使用静态常量定义可组合的选项。

```swift
struct FilePermissions: OptionSet {
    let rawValue: Int
    
    static let read    = FilePermissions(rawValue: 1 << 0)
    static let write   = FilePermissions(rawValue: 1 << 1)
    static let execute = FilePermissions(rawValue: 1 << 2)
    
    static let readWrite: FilePermissions = [.read, .write]
}

let permissions: FilePermissions = [.read, .write]
print(permissions.contains(.execute)) // false
print(permissions.contains(.read))    // true
```

## RawRepresentable

**说明**：允许类型与一个原始值（raw value）之间互相转换。枚举可以自动获得此协议的实现。

```swift
enum Direction: String, RawRepresentable {
    case north = "N"
    case south = "S"
    case east  = "E"
    case west  = "W"
}

let d = Direction(rawValue: "N") // .north
print(d?.rawValue ?? "") // "N"
```

## Any & AnyObject & Never

**说明**：
- **`Any`**：可以表示任何类型的实例，包括值类型和函数类型。
- **`AnyObject`**：只能表示类类型的实例，常用于泛型约束或协议中的 `where` 子句。
- **`Never`**：不可实例化的类型，表示函数不会正常返回（如 `fatalError()` 的返回类型）。

```swift
// Any 可以存放任何值
var anything: Any = 42
anything = "Hello"
anything = { (a: Int, b: Int) -> Int in a + b }

// AnyObject 限定为类
class Dog {}
let pet: AnyObject = Dog()

// Never 用于标记不会返回的函数
func crash() -> Never {
    fatalError("意料之外的错误")
}
```

---

本附录列出的协议和类型覆盖了 Swift 标准库中最常见、最核心的抽象。掌握它们，就等于掌握了 Swift 类型系统的「半壁江山」。在实际编码中，建议优先使用标准库协议而非自定义接口，这样代码更具通用性和互操作性。
