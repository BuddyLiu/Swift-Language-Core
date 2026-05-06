# 第1章 类型与变量

## 1.1 一切皆类型

在 Swift 的世界里，"一切皆类型"不仅仅是一句口号，它是这门语言的设计哲学。当你第一次接触 Swift 时，最需要转变的思维方式就是：**万物皆有其类型**。

整数有类型，字符串有类型，布尔值有类型，数组和字典有类型。函数有类型，闭包有类型。甚至错误、可选值、元组，全都有类型。

```swift
let age: Int = 28
let name: String = "Swift"
let isActive: Bool = true
let scores: [Int] = [85, 92, 78]
let user: [String: String] = ["name": "Tom", "city": "Beijing"]
```

这些基础类型组成了我们日常编码的基石。但 Swift 的类型系统远不止于此——**函数本身也是类型**。你可以把函数赋值给变量，作为参数传递，或者作为返回值返回。这意味着函数在 Swift 中是一等公民（first-class citizen）。

```swift
// 函数是类型
func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}

// 将函数赋值给变量
let operation: (Int, Int) -> Int = add
let result = operation(3, 5)  // 8

// 闭包也是类型
let multiply = { (a: Int, b: Int) -> Int in
    return a * b
}
print(multiply(4, 5))  // 20
```

这种设计让 Swift 具备了强大的表达能力。你可以编写高阶函数，创建灵活的 API，甚至构建自己的领域特定语言（DSL）。**类型安全**是 Swift 最核心的特性之一——一旦一个变量的类型确定，你就不能赋给它其他类型的值。这虽然在最初可能让你觉得有些约束，但实际上它消除了大量潜在的运行时错误。

## 1.2 类型推断：写得更少，做得更多

你可能注意到了，前面的例子中很多变量声明并没有显式写出类型。这是因为 Swift 拥有强大的**类型推断**（Type Inference）机制。

```swift
let count = 42          // 编译器自动推断为 Int
let price = 19.99       // 推断为 Double
let message = "Hello"   // 推断为 String
let isDone = false      // 推断为 Bool
let items = [1, 2, 3]   // 推断为 [Int]
```

编译器会根据你赋的初始值自动推导出变量的类型。这不仅减少了代码量，也让代码更加清晰易读。当编译器能明确推断时，你完全不需要书写类型注解。

那什么时候需要显式声明类型呢？主要有以下几种场景：

1. **声明变量但没有初始值时**
2. **希望类型与默认推断不同时**（比如希望一个整数是 `Int16` 而非 `Int`）
3. **字面量可能有歧义时**

```swift
// 没有初始值，必须声明类型
var username: String

// 指定精确类型
let smallValue: Int16 = 100

// 浮点数字面量默认是 Double，如果需要 Float
let precise: Float = 3.14
```

类型推断让 Swift 既具备了静态类型的安全性，又拥有了动态类型语言的简洁感。它是 Swift 类型系统中让你"写得更少，做得更多"的利器。

## 1.3 值类型与引用类型的基本区分

Swift 中一个极其重要的概念区分就是**值类型**（Value Type）和**引用类型**（Reference Type）。这个区分深刻影响了代码的行为、性能和安全性。

### 结构体（值类型）

结构体（`struct`）是值类型。当你把一个结构体实例赋值给另一个变量，或者传递给函数时，**它会被复制**。这意味着两个变量各自拥有独立的数据副本。

```swift
struct Point {
    var x: Double
    var y: Double
}

var p1 = Point(x: 10, y: 20)
var p2 = p1        // p1 被复制给 p2

p2.x = 99          // 只修改 p2
print(p1.x)        // 10 — p1 不受影响
print(p2.x)        // 99
```

### 类（引用类型）

类（`class`）是引用类型。当你把一个类的实例赋值给另一个变量时，**两个变量指向同一个内存地址**。修改其中一个，另一个也会变化。

```swift
class Person {
    var name: String
    init(name: String) { self.name = name }
}

let person1 = Person(name: "Alice")
let person2 = person1   // person2 指向同一个实例

person2.name = "Bob"
print(person1.name)     // "Bob" — 被修改了！
print(person2.name)     // "Bob"
```

### 赋值行为差异的核心

值类型和引用类型的根本区别在于赋值时的行为：

| 特征 | 值类型（struct） | 引用类型（class） |
|------|-----------------|-------------------|
| 赋值时 | 复制数据 | 复制引用（指针） |
| 存储位置 | 栈（Stack）为主 | 堆（Heap） |
| 修改独立性 | 完全独立 | 共享同一份数据 |
| 线程安全 | 天然更安全 | 需额外注意 |

Swift 中基本类型都是结构体——这正是下一节要深入探讨的内容。理解这个区别，是写出正确、高效 Swift 代码的基础。

### 如何选择？

苹果官方的建议是：**默认使用结构体**。只有在需要引用语义、继承、或者与 Objective-C 互操作时，才使用类。这个建议源于值类型带来的安全性和可预测性。

## 1.4 常量的力量

Swift 引入了一个在其他语言中不太常见的关键字——`let`，用于声明**常量**。与之对应的是 `var`，用于声明**变量**。

```swift
let maxRetries = 3     // 常量：不可修改
var currentCount = 0   // 变量：可以修改
currentCount += 1      // ✅ 正确
// maxRetries = 5      // ❌ 编译错误
```

这看起来似乎只是一个小小的语法差异，但它对代码质量的影响是深远的。

### 不变的承诺

当你用 `let` 声明一个值时，你不仅告诉编译器这个值不会改变，更重要的是，你**告诉阅读代码的人**：这个值在整个生命周期中保持不变。这消除了心智负担——读者不需要追踪这个值是否会在后续代码中被修改。

```swift
let pi = 3.14159
let appName = "MyApp"
let apiBaseURL = "https://api.example.com"
```

像上述这些值，它们从创建到销毁都不应该改变。用 `let` 声明它们，编译器会确保这一点。

### let 与安全

尽可能使用 `let` 能够让代码更安全。想想看，有多少 bug 是因为某个变量在不该被修改的地方被意外修改了？`let` 从根本上杜绝了这类问题。

```swift
// 好的做法：能用 let 就用 let
let userID = "abc123"
let isVIP = true

// 只有真正需要变化时才用 var
var score = 0
```

Swift 社区的推荐实践是：**始终优先使用 `let`，只在明确需要改变时才改用 `var`**。Xcode 甚至会在你声明了一个 `var` 但从未修改时给出警告，提示你改为 `let`。

### 引用类型的 let

需要特别注意的是，`let` 对于引用类型的语义：它保证的是**引用本身不变**，而不是被引用的对象不可变。

```swift
let person = Person(name: "Alice")
// person = Person(name: "Bob")  // ❌ 不能改变 person 的指向
person.name = "Bob"              // ✅ 但可以修改对象的属性
```

如果希望对象本身也不可变，需要使用值类型（`struct`），或者对类属性使用 `let` 声明。

## 1.5 基本类型的内部视角

你可能来自其他编程语言，在那里 `int`、`string`、`array` 等基本类型有着特殊地位——它们可能是"原始类型"（primitive type），与用户自定义的类型地位不同。

但在 Swift 中，**一切皆结构体**。基本类型全都是定义在标准库中的结构体。

### Int 是结构体

```swift
// Swift 标准库中 Int 的简化定义
public struct Int: SignedInteger, Comparable, Equatable {
    public var _value: Builtin.Int64
    // ... 大量方法和扩展
}
```

`Int` 是一个结构体，这意味着它是一个值类型。它遵循了 `SignedInteger`、`Comparable`、`Equatable` 等协议。这就是为什么你可以调用 `42.description`、`(-5).magnitude` 等方法——因为 Int 和其他类型一样，拥有方法和属性。

```swift
let number = 42
print(number.description)           // "42"
print(number.isMultiple(of: 7))     // true
print(Int.max)                      // 9223372036854775807
print(Int.min)                      // -9223372036854775808
```

### String 是结构体

`String` 同样是结构体，这与其他语言（如 Objective-C 的 `NSString` 是类）有显著不同。

```swift
let greeting = "Hello"
print(greeting.count)               // 5
print(greeting.hasPrefix("He"))     // true
print(greeting.uppercased())        // "HELLO"

// String 是集合，可以遍历
for char in greeting {
    print(char)  // H, e, l, l, o
}
```

作为值类型，字符串在赋值时会被复制。但 Swift 通过**写时复制**（Copy-on-Write）优化了性能——只有在真正需要修改时才会执行实际的复制操作。

### Array 和 Dictionary 也是结构体

```swift
var numbers = [1, 2, 3]
numbers.append(4)                   // 调用结构体的方法
print(numbers.count)                // 4
print(numbers.first ?? 0)           // 1

var user = ["name": "Tom", "age": "28"]
user["city"] = "Beijing"            // 添加键值对
```

`Array` 和 `Dictionary` 同样是定义在标准库中的结构体，它们是**泛型结构体**：

```swift
// 标准库中的简化定义
public struct Array<Element>: Collection, MutableCollection { ... }
public struct Dictionary<Key, Value>: Collection where Key: Hashable { ... }
```

### 值类型的意义

所有基本类型都是结构体，这意味着：

1. **不受继承污染**：没有来自基类的冗余属性
2. **赋值即复制**：行为可预测，没有意外共享
3. **线程安全**：每个线程持有自己的副本
4. **性能可预期**：栈分配为主，减少堆压力

```swift
var a = [1, 2, 3]
var b = a       // 复制整个数组
b.append(4)     // 只修改 b
print(a.count)  // 3 — a 不受影响
```

这种设计确保了代码行为的可预测性。当你传递一个数组给函数时，你完全不需要担心函数内部是否会修改原始数据——因为函数得到的是数据的副本。

### 小结

本章我们学习了 Swift 类型系统的基础。一切皆类型的理念让语言高度统一；类型推断带来了简洁性；值类型和引用类型的区分是理解数据行为的关键；常量的力量来自编译器的强制保证；而所有基本类型都是结构体的事实，则揭示了 Swift 对安全性和可预测性的极致追求。

在下一章中，我们将探讨 Swift 中最具特色的特性之一——可选类型，它将根本性地改变你处理"空值"的方式。
