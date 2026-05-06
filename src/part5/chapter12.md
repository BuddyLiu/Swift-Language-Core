# 第12章 函数式内核

函数式编程（Functional Programming）是一种以函数为核心组织代码的编程范式。Swift 虽然不是一门纯函数式语言，但它从 Haskell、OCaml 等语言中借鉴了大量函数式特性，形成了一个优雅的"函数式内核"。本章将深入探讨这些特性，并展示如何在实际项目中使用它们写出更简洁、更可测试的代码。

## 12.1 函数是一等公民：函数作为变量、参数、返回值

"一等公民"（First-class citizen）意味着函数可以像其他值一样被使用：赋值给变量、作为参数传递、作为返回值返回。这是函数式编程的基础。

### 函数赋值给变量

```swift
// 定义一个普通函数
func add(_ a: Int, _ b: Int) -> Int {
    return a + b
}

// 将函数赋值给变量
let mathOperation: (Int, Int) -> Int = add
let result = mathOperation(3, 4)  // 7

// 使用闭包语法直接赋值
let multiply: (Int, Int) -> Int = { $0 * $1 }
print(multiply(3, 4))  // 12
```

### 函数作为参数

这是函数式编程中最常见的模式——将行为参数化：

```swift
func applyOperation(_ a: Int, _ b: Int, operation: (Int, Int) -> Int) -> Int {
    return operation(a, b)
}

// 传递不同函数实现不同行为
let sum = applyOperation(10, 5, operation: +)   // 15
let difference = applyOperation(10, 5, operation: -)  // 5
let product = applyOperation(10, 5, operation: *)  // 50

// 传递闭包
let custom = applyOperation(10, 5) { x, y in
    return (x * x) + (y * y)
}  // 10² + 5² = 125

// 排序中的行为参数化
let numbers = [3, 1, 4, 1, 5, 9, 2, 6]
let sortedAscending = numbers.sorted(by: <)
let sortedDescending = numbers.sorted(by: >)
let customSorted = numbers.sorted { $0 % 3 < $1 % 3 }
```

### 函数作为返回值

函数作为返回值让我们能够创建"函数工厂"——根据输入生成特定行为的函数：

```swift
// 根据比较条件生成比较器
func makeComparator(ascending: Bool) -> (Int, Int) -> Bool {
    return ascending ? { $0 < $1 } : { $0 > $1 }
}

let ascendingComparator = makeComparator(ascending: true)
let descendingComparator = makeComparator(ascending: false)

print(ascendingComparator(3, 5))   // true
print(descendingComparator(3, 5))  // false

// 更实用的例子：创建格式化函数工厂
func makeFormatter(prefix: String, suffix: String) -> (String) -> String {
    return { value in
        return "\(prefix)\(value)\(suffix)"
    }
}

let withBrackets = makeFormatter(prefix: "[", suffix: "]")
let withQuotes = makeFormatter(prefix: "\"", suffix: "\"")

print(withBrackets("Swift"))  // [Swift]
print(withQuotes("Hello"))    // "Hello"
```

### 捕获上下文（闭包）

闭包可以捕获其定义时的上下文变量，这是函数式编程中"闭包"（closure）概念的核心：

```swift
func makeCounter() -> () -> Int {
    var count = 0
    return {
        count += 1
        return count
    }
}

let counter1 = makeCounter()
let counter2 = makeCounter()

print(counter1())  // 1
print(counter1())  // 2
print(counter1())  // 3
print(counter2())  // 1  — 独立的计数器
```

## 12.2 高阶函数：map、flatMap、filter、reduce

高阶函数（Higher-order functions）是接受函数作为参数或返回函数的函数。Swift 标准库中最常见的高阶函数是 `map`、`filter`、`reduce` 和 `flatMap`。

### map：变换集合中的每个元素

`map` 对集合中的每个元素应用一个变换函数，返回一个包含变换后元素的新集合：

```swift
let numbers = [1, 2, 3, 4, 5]

// 传统方式
var squaredTraditional: [Int] = []
for n in numbers {
    squaredTraditional.append(n * n)
}

// map 方式
let squared = numbers.map { $0 * $0 }
print(squared)  // [1, 4, 9, 16, 25]

// 更多 map 示例
let names = ["alice", "bob", "charlie"]
let capitalized = names.map { $0.capitalized }
print(capitalized)  // ["Alice", "Bob", "Charlie"]

// 字典的 map
let scores = ["Alice": 85, "Bob": 92, "Charlie": 78]
let descriptions = scores.map { (name, score) in
    return "\(name) 得了 \(score) 分"
}
print(descriptions)  // ["Alice 得了 85 分", "Bob 得了 92 分", "Charlie 得了 78 分"]

// Optional 的 map
let optionalNumber: Int? = 5
let doubled = optionalNumber.map { $0 * 2 }
print(doubled as Any)  // Optional(10)
```

### compactMap：过滤 nil

`compactMap` 是 `map` 的一个变体，它会自动过滤掉变换结果为 `nil` 的元素：

```swift
let strings = ["1", "2", "three", "4", "five", "6"]

// 传统方式
var parsedTraditional: [Int] = []
for s in strings {
    if let n = Int(s) {
        parsedTraditional.append(n)
    }
}

// compactMap 方式
let parsed = strings.compactMap { Int($0) }
print(parsed)  // [1, 2, 4, 6]
```

### filter：按条件筛选

`filter` 根据一个返回 `Bool` 的闭包来筛选集合中的元素：

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

let evenNumbers = numbers.filter { $0.isMultiple(of: 2) }
print(evenNumbers)  // [2, 4, 6, 8, 10]

let bigNumbers = numbers.filter { $0 > 5 }
print(bigNumbers)  // [6, 7, 8, 9, 10]

// 链式调用
let result = numbers
    .filter { $0 > 3 }        // [4, 5, 6, 7, 8, 9, 10]
    .filter { $0.isMultiple(of: 2) }  // [4, 6, 8, 10]
    .map { $0 * $0 }           // [16, 36, 64, 100]
print(result)  // [16, 36, 64, 100]
```

### reduce：将集合归约为单一值

`reduce` 将集合中的元素合并为一个单一值：

```swift
let numbers = [1, 2, 3, 4, 5]

// 求和 — reduce 接受初始值和合并闭包
let sum = numbers.reduce(0) { $0 + $1 }
print(sum)  // 15

// 更简洁的写法
let sumShort = numbers.reduce(0, +)
print(sumShort)  // 15

// 最大值
let maxValue = numbers.reduce(Int.min) { current, next in
    return current > next ? current : next
}
print(maxValue)  // 5

// 字符串拼接
let words = ["Swift", "是", "一门", "强大的", "语言"]
let sentence = words.reduce("") { $0 + $1 }
print(sentence)  // Swift是一门强大的语言

// 统计词频
let wordList = ["apple", "banana", "apple", "orange", "banana", "apple"]
let wordFrequency = wordList.reduce(into: [:]) { counts, word in
    counts[word, default: 0] += 1
}
print(wordFrequency)  // ["apple": 3, "banana": 2, "orange": 1]
```

### flatMap：展平嵌套集合

`flatMap` 将嵌套的集合展平为一层：

```swift
let nested = [[1, 2, 3], [4, 5], [6, 7, 8, 9]]

let flattened = nested.flatMap { $0 }
print(flattened)  // [1, 2, 3, 4, 5, 6, 7, 8, 9]

// 实际场景：获取所有学生的所有课程
struct Student {
    let name: String
    let courses: [String]
}

let students = [
    Student(name: "Alice", courses: ["Math", "Physics"]),
    Student(name: "Bob", courses: ["Chemistry", "Biology", "Physics"]),
    Student(name: "Charlie", courses: ["History"])
]

let allCourses = students.flatMap { $0.courses }
print(Set(allCourses))
// 输出：["Math", "Biology", "Chemistry", "Physics", "History"]
```

### 链式调用的实战示例

将上述高阶函数组合使用，可以写出极具表达力的代码：

```swift
struct Transaction {
    let amount: Double
    let category: String
    let isPending: Bool
}

let transactions = [
    Transaction(amount: 49.99, category: "餐饮", isPending: false),
    Transaction(amount: 299.00, category: "购物", isPending: true),
    Transaction(amount: 15.50, category: "交通", isPending: false),
    Transaction(amount: 899.00, category: "数码", isPending: false),
    Transaction(amount: 35.00, category: "餐饮", isPending: false),
    Transaction(amount: 199.00, category: "购物", isPending: false),
]

// 需求：已完成交易中，按类别统计总金额，取前两名
let categorySpending = transactions
    .filter { !$0.isPending }           // 只取已完成交易
    .reduce(into: [:]) { result, t in    // 按类别汇总
        result[t.category, default: 0.0] += t.amount
    }
    .sorted { $0.value > $1.value }      // 按金额降序排列
    .prefix(2)                           // 取前两名
    .map { "\($0.key): ¥\($0.value)" }   // 格式化输出

print(categorySpending)
// 可能的输出：["数码: ¥899.0", "购物: ¥199.0"]
```

## 12.3 函数的类型与可组合性

### 函数类型

在 Swift 中，每个函数都有一个具体的类型，由参数类型和返回类型决定：

```swift
// (Int, Int) -> Int
func add(_ a: Int, _ b: Int) -> Int { a + b }

// (String) -> Void
func greet(_ name: String) { print("Hello, \(name)!") }

// () -> () 或 () -> Void
func doNothing() {}

// (Int) -> (Int) -> Int  — 柯里化形式
func addCurried(_ a: Int) -> (Int) -> Int {
    return { b in a + b }
}
```

### 函数的组合

函数组合（Function Composition）是将多个小函数组合成一个新函数的技术：

```swift
// 手工组合
func compose<A, B, C>(_ f: @escaping (A) -> B, _ g: @escaping (B) -> C) -> (A) -> C {
    return { x in g(f(x)) }
}

func double(_ x: Int) -> Int { x * 2 }
func addOne(_ x: Int) -> Int { x + 1 }

let doubleThenAddOne = compose(double, addOne)
print(doubleThenAddOne(5))  // double(5)=10, addOne(10)=11

// 自定义组合运算符
infix operator >>>: CompositionPrecedence
precedencegroup CompositionPrecedence {
    associativity: left
    higherThan: DefaultPrecedence
}

func >>><A, B, C>(_ f: @escaping (A) -> B, _ g: @escaping (B) -> C) -> (A) -> C {
    return { x in g(f(x)) }
}

let pipeline = double >>> addOne >>> String.init
print(pipeline(5))  // "11"
```

### 点自由风格

当函数组合成为主导时，我们可以写出"点自由风格"（Point-free style）的代码——即不显式提到操作的数据：

```swift
// 非点自由风格
let numbers = [1, 2, 3, 4, 5]
let squaredEvenSum = numbers
    .filter { $0.isMultiple(of: 2) }  // 显式提到 $0
    .map { $0 * $0 }                  // 显式提到 $0
    .reduce(0, +)

// 更接近点自由风格——使用函数引用
func isEven(_ n: Int) -> Bool { n.isMultiple(of: 2) }
func square(_ n: Int) -> Int { n * n }

let squaredEvenSum2 = numbers
    .filter(isEven)
    .map(square)
    .reduce(0, +)

print(squaredEvenSum2)  // 20
```

## 12.4 柯里化与部分应用

### 柯里化

柯里化（Currying）是将一个接受多个参数的函数转换为一系列接受单一参数的函数的技术：

```swift
// 普通的多参数函数
func multiply(_ a: Int, _ b: Int) -> Int {
    return a * b
}

// 柯里化版本
func curriedMultiply(_ a: Int) -> (Int) -> Int {
    return { b in a * b }
}

let double = curriedMultiply(2)
let triple = curriedMultiply(3)

print(double(5))  // 10
print(triple(5))  // 15

// 三个参数的柯里化
func curriedAdd(_ a: Int) -> (Int) -> (Int) -> Int {
    return { b in
        return { c in
            a + b + c
        }
    }
}

let result = curriedAdd(1)(2)(3)
print(result)  // 6

// 使用部分应用
let addFive = curriedAdd(5)
let addFiveAndThree = addFive(3)
print(addFiveAndThree(2))  // 5 + 3 + 2 = 10
```

### 部分应用

部分应用（Partial Application）是指固定函数的一部分参数，产生一个接受剩余参数的新函数：

```swift
// 手动实现部分应用
func partial<A, B, C>(_ f: @escaping (A, B) -> C, _ a: A) -> (B) -> C {
    return { b in f(a, b) }
}

func power(base: Double, exponent: Double) -> Double {
    return pow(base, exponent)
}

let square2 = partial(power, 2.0)
let cube = partial(power, 3.0)

print(square2(3))  // 8.0  (2³)
print(cube(3))     // 27.0 (3³)

// 更实用的例子——Text 格式化
func formatText(_ text: String, prefix: String, suffix: String) -> String {
    return "\(prefix)\(text)\(suffix)"
}

let wrapInTag = partial(partial(formatText, prefix: "<b>"), suffix: "</b>")
// 但上面的写法冗长。更好的方式是为特定场景创建专用函数：

func makeTagWrapper(tag: String) -> (String) -> String {
    return { text in
        return "<\(tag)>\(text)</\(tag)>"
    }
}

let boldWrapper = makeTagWrapper(tag: "b")
let italicWrapper = makeTagWrapper(tag: "i")

print(boldWrapper("Hello"))   // <b>Hello</b>
print(italicWrapper("World")) // <i>World</i>
```

### Swift 原生方法引用

Swift 的方法引用语法天然支持柯里化风格的调用：

```swift
import Foundation

struct Person {
    let name: String
    let age: Int
}

let people = [
    Person(name: "Alice", age: 30),
    Person(name: "Bob", age: 25),
    Person(name: "Charlie", age: 35)
]

// Swift 方法的柯里化特征
// sorted(by:) 可以看作是 (Self) -> ((Element, Element) -> Bool) -> [Element]
let ageAscending = people.sorted { $0.age < $1.age }
let nameDescending = people.sorted { $0.name > $1.name }

// Key path + 高阶函数的组合
let names = people.map(\.name)           // ["Alice", "Bob", "Charlie"]
let ages = people.map(\.age)             // [30, 25, 35]
let totalAge = people.reduce(0) { $0 + $1.age }  // 90
let averageAge = Double(totalAge) / Double(people.count)  // 30.0
```

## 12.5 不可变性与值类型：函数式思维的基石

函数式编程的核心理念之一是不可变性（Immutability）。Swift 通过值类型（`struct`、`enum`）和 `let` 关键字为我们提供了强大的不可变性支持。

### let 与 var 的选择

```swift
// 函数式风格：尽可能使用 let
struct Point {
    let x: Double
    let y: Double

    // "修改"操作返回新实例，而非修改自身
    func moveBy(dx: Double, dy: Double) -> Point {
        return Point(x: x + dx, y: y + dy)
    }
}

let p1 = Point(x: 1.0, y: 2.0)
let p2 = p1.moveBy(dx: 3.0, dy: 4.0)
// p1 保持不变: (1.0, 2.0)
// p2: (4.0, 6.0)

// 对比可变风格
class MutablePoint {
    var x: Double
    var y: Double
    
    init(x: Double, y: Double) {
        self.x = x
        self.y = y
    }
    
    func moveBy(dx: Double, dy: Double) {
        x += dx
        y += dy
    }
}
```

### 值类型的优势

值类型（struct、enum、tuple）在赋值和传参时会被复制，这天然隔离了副作用：

```swift
struct Account {
    let name: String
    var balance: Double
    
    // 函数式风格的"更新"操作
    func deposit(amount: Double) -> Account {
        return Account(name: name, balance: balance + amount)
    }
    
    func withdraw(amount: Double) -> Result<Account, String> {
        guard balance >= amount else {
            return .failure("余额不足")
        }
        return .success(Account(name: name, balance: balance - amount))
    }
}

// 使用起来像函数式链
let initial = Account(name: "Alice", balance: 1000)
let result = initial
    .deposit(amount: 500)    // 余额：1500
    .withdraw(amount: 200)   // 成功，余额：1300
    .map { $0.withdraw(amount: 1600) }  // 失败

switch result {
case .success(let account):
    print("最终余额：\(account.balance)")
case .failure(let error):
    print("操作失败：\(error)")
}
```

### 使用不可变数据结构

在函数式编程中，我们经常对集合进行"纯"变换，而不是就地修改：

```swift
// 不可变的购物车
struct ShoppingCart {
    private var items: [String: Int]  // 商品名：数量
    
    init(items: [String: Int] = [:]) {
        self.items = items
    }
    
    // 每次修改都返回新实例
    func adding(item: String, quantity: Int = 1) -> ShoppingCart {
        var newItems = items
        newItems[item, default: 0] += quantity
        return ShoppingCart(items: newItems)
    }
    
    func removing(item: String) -> ShoppingCart {
        var newItems = items
        newItems.removeValue(forKey: item)
        return ShoppingCart(items: newItems)
    }
    
    func updatingQuantity(item: String, to quantity: Int) -> ShoppingCart {
        var newItems = items
        newItems[item] = quantity
        return ShoppingCart(items: newItems)
    }
    
    var totalItems: Int {
        return items.values.reduce(0, +)
    }
}

let emptyCart = ShoppingCart()
let cartWithApple = emptyCart.adding(item: "苹果", quantity: 3)
let cartWithBanana = cartWithApple.adding(item: "香蕉", quantity: 2)
let finalCart = cartWithBanana.updatingQuantity(item: "苹果", to: 5)

// emptyCart, cartWithApple, cartWithBanana 都保持不可变
print(emptyCart.totalItems)    // 0
print(cartWithApple.totalItems)     // 3
print(cartWithBanana.totalItems)    // 5
print(finalCart.totalItems)         // 7
```

### 纯函数：无副作用，可预测

纯函数（Pure Function）满足两个条件：1）相同的输入始终产生相同的输出；2）没有副作用（不修改外部状态，不进行 I/O 操作）。

```swift
// ✅ 纯函数
func pureAdd(_ a: Int, _ b: Int) -> Int {
    return a + b
}

func factorial(_ n: Int) -> Int {
    return n <= 1 ? 1 : n * factorial(n - 1)
}

// ❌ 非纯函数：依赖外部状态
var multiplier = 2
func impureMultiply(_ x: Int) -> Int {
    return x * multiplier  // 依赖可变的外部变量
}

// ❌ 非纯函数：有副作用
func impureLog(_ message: String) {
    print(message)  // 副作用：控制台输出
}

// 实际应用：通过纯函数的组合实现复杂逻辑
func processTransactions(transactions: [Transaction]) -> [String: Double] {
    // 一系列的纯变换
    return transactions
        .filter { !$0.isPending }                    // 过滤
        .reduce(into: [:]) { result, t in            // 聚合
            result[t.category, default: 0.0] += t.amount
        }
}

// 这个函数是纯函数——相同的 transactions 输入始终得到相同输出
// 可以轻松测试、缓存、并行执行
```

### 函数式思维的实践原则

以下是编写函数式风格 Swift 代码的实用原则：

1. **优先使用 `let` 而非 `var`**：尽可能地声明不可变值
2. **优先使用 `struct` 而非 `class`**：值类型天然隔离副作用
3. **避免共享可变状态**：函数不修改外部变量
4. **使用 `map`/`filter`/`reduce` 代替循环**：声明式而非命令式
5. **从函数返回新值而非修改传入的参数**：保留原始数据不变
6. **将副作用推到系统边界**：核心逻辑保持纯函数，I/O 放在边缘层

```swift
// 遵循函数式原则的设计示例
struct OrderProcessor {
    // 核心逻辑：纯函数
    static func calculateDiscount(
        for orderTotal: Double,
        customerSince: Int
    ) -> Double {
        let loyaltyDiscount = min(Double(customerSince) * 0.01, 0.2)
        let volumeDiscount = orderTotal > 1000 ? 0.1 : 0.0
        return max(loyaltyDiscount, volumeDiscount)
    }
    
    // 核心逻辑：纯函数
    static func applyDiscount(
        to orderTotal: Double,
        discount: Double
    ) -> Double {
        return orderTotal * (1 - discount)
    }
    
    // 核心逻辑：纯函数
    static func calculateTax(
        on subtotal: Double,
        taxRate: Double
    ) -> Double {
        return subtotal * taxRate
    }
    
    // 边缘层：组合纯函数 + 副作用
    static func processOrder(
        items: [CartItem],
        customer: Customer
    ) -> OrderResult {
        let subtotal = items
            .map { $0.price * Double($0.quantity) }
            .reduce(0, +)
        
        let discount = calculateDiscount(
            for: subtotal,
            customerSince: customer.membershipYears
        )
        
        let afterDiscount = applyDiscount(to: subtotal, discount: discount)
        let tax = calculateTax(on: afterDiscount, taxRate: 0.08)
        let total = afterDiscount + tax
        
        // 在最后一步进行副作用操作
        return saveOrder(items: items, total: total)
    }
}
```

## 小结

本章探索了 Swift 的函数式编程内核。从函数作为一等公民开始，我们了解了如何将行为参数化、如何通过高阶函数声明式地操作集合。函数的可组合性让我们可以将小函数拼装成复杂的处理管道，而柯里化和部分应用则提供了参数复用的灵活手段。

不可变性和值类型是函数式思维的基石，它们帮助我们写出更安全、更可预测、更易测试的代码。在实际项目中，不必追求纯函数式风格，而是有选择地应用这些原则——在核心业务逻辑中保持纯函数，在系统边界处处理副作用。这种"函数式内核 + 命令式外壳"的混合风格，往往是最实用、最有效的 Swift 开发方式。
