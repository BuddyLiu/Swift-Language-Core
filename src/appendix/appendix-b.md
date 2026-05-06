# 附录B Swift 演进历史与核心版本变化

Swift 自 2014 年发布以来，经历了前所未有的语言演进速度。本附录按时间线梳理每个大版本的核心变化，帮助读者理解 Swift 语言设计理念的演变脉络。

## Swift 1.0 — 2014 年 6 月

**WWDC 发布**：Swift 在 2014 年 WWDC 上首次亮相，震惊了整个开发者社区。这是 Apple 抛弃 Objective-C 的「历史包袱」、从头设计现代语言的雄心之作。

**核心特性**：
- 基础语法：类型推断、闭包、元组、泛型
- 可选值（Optional）：`?` 和 `!` 语法，安全地处理 nil
- ARC（Automatic Reference Counting）：自动内存管理，无垃圾回收
- Playground：交互式编程环境，实时查看代码执行结果
- 与 Objective-C 互操作：可桥接 Foundation、UIKit 等框架

```swift
// Swift 1.0 风格的代码
let array = [1, 2, 3]
let doubled = array.map({ $0 * 2 }) // 闭包语法
println(doubled) // 注意：当时使用 println 而非 print
```

## Swift 2.0 — 2015 年 9 月

**核心特性**：
- **错误处理**：引入 `throws`、`try`、`catch`、`defer` 关键字
- **`guard` 语句**：提前退出，减少嵌套
- **协议扩展（Protocol Extensions）**：可为协议提供默认实现
- **`#available`**：平台版本检查
- **`do` 语句**：作用域控制

```swift
// Swift 2.0 的错误处理
enum FileError: Error {
    case notFound
}

func readFile(named name: String) throws -> String {
    guard !name.isEmpty else {
        throw FileError.notFound
    }
    return "内容"
}

defer {
    print("清理资源") // 作用域结束时执行
}
```

## Swift 3.0 — 2016 年 9 月

**核心特性**：这是 Swift 历史上最「阵痛」的一次变更——**全面 API 命名规范改革**。Apple 移除了大量 Objective-C 风格的命名残留，改为更自然的英文短语。

**主要变化**：
- `dispatch_async` → `DispatchQueue.main.async {}`
- `NSArray`、`NSDictionary` 桥接行为改变
- Foundation 类型去掉 `NS` 前缀（如 `URL` 而非 `NSURL`）
- 函数参数标签规则统一
- `++` 和 `--` 操作符被废弃

```swift
// Swift 3.0 的 API 风格
let url = URL(string: "https://apple.com")!
let task = URLSession.shared.dataTask(with: url) { data, _, error in
    // 回调风格
}
```

## Swift 4.0 — 2017 年 9 月

**核心特性**：
- **`Codable`**：`Encodable` + `Decodable` 协议，极大简化 JSON 序列化
- **字符串改进**：`Substring` 类型、多行字符串字面量 `"""`
- **`KeyPath`**：类型安全的键路径
- **归档序列化**：`NSKeyedArchiver` 的 Swift 原生替代
- **关联类型约束改进**：`where` 子句更灵活

```swift
// Swift 4.0 的 Codable
struct Product: Codable {
    var name: String
    var price: Double
}

let json = """
{"name": "MacBook", "price": 12999.0}
""".data(using: .utf8)!
let product = try JSONDecoder().decode(Product.self, from: json)

// 多行字符串
let poem = """
床前明月光
疑是地上霜
举头望明月
低头思故乡
"""
```

## Swift 5.0 — 2019 年 3 月

**核心特性**：Swift 5 是里程碑式的版本，最重要的成就是 **ABI（Application Binary Interface）稳定性**。

**主要变化**：
- **ABI 稳定**：Swift 运行时版本与操作系统绑定，不再需要嵌入 Swift 标准库
- **原始字符串**：`#"..."#` 语法，减少转义烦恼
- **`Result` 类型**：`Success` / `Failure` 枚举
- **`is` 和 `as?` 改进**：枚举 case 模式匹配增强
- **标准库扩展**：`compactMap`、`first(where:)` 等

```swift
// Swift 5.0 的原始字符串
let regex = #"\d{3}-\d{4}"#
let text = #"换行符是 \n，但这里按字面处理"#

// Result 类型
let result: Result<Int, Error> = .success(42)
switch result {
case .success(let value):
    print(value)
case .failure(let error):
    print(error)
}
```

## Swift 5.5 — 2021 年 9 月

**核心特性**：引入了 Swift 并发模型，这是 Swift 诞生以来最重大的语言特性新增。

**主要变化**：
- **`async/await`**：异步函数的全新写法
- **`Actor`**：保护可变状态免受数据竞争
- **`Sendable`**：跨并发域传递的类型安全约束
- **结构化并发**：`Task`、`TaskGroup`、`async let`
- **全局 Actor**：`@MainActor` 注解

```swift
// Swift 5.5 的 async/await
actor BankAccount {
    var balance: Double = 0
    
    func deposit(_ amount: Double) {
        balance += amount
    }
    
    func withdraw(_ amount: Double) -> Bool {
        if balance >= amount {
            balance -= amount
            return true
        }
        return false
    }
}

let account = BankAccount()
await account.deposit(100)
let success = await account.withdraw(50)
```

## Swift 5.7+ 到最新版本

### Swift 5.7 (2022)
- **不透明类型增强**：`some` 可用于参数位置
- **`any` 关键字**：显式标记存在类型（existential type）
- **正则表达式字面量**：`/pattern/` 语法
- **可选绑定简化**：`if let x` 无需 `=` 写法

```swift
// Swift 5.7 的简化可选绑定
let name: String? = "Alice"
if let name { // 等价于 if let name = name
    print("Hello, \(name)")
}

// any 关键字
let anyShape: any Shape = Circle()

// 正则字面量
let regex = /(\d+)-(\d+)/
```

### Swift 5.9 (2023)
- **参数包（Parameter Packs）**：泛型可变参数
- **宏（Macros）**：编译期元编程
- **所有权（Ownership）**：`borrowing` / `consuming` 关键字

### Swift 6.0 (预计)
Swift 6 将迎来**严格并发检查（Strict Concurrency Checking）** 成为默认行为，这意味着：
- 所有并发相关的警告将变为错误
- `Sendable` 检查更加严格
- Actor 隔离规则更完备

---

### 演进小结

| 版本 | 年份 | 核心主题 |
|------|------|----------|
| 1.0 | 2014 | 语言诞生，可选值，ARC |
| 2.0 | 2015 | 错误处理，guard，协议扩展 |
| 3.0 | 2016 | API 命名规范改革 |
| 4.0 | 2017 | Codable，字符串改进 |
| 5.0 | 2019 | ABI 稳定，Result 类型 |
| 5.5 | 2021 | async/await，Actor |
| 5.7-5.9 | 2022-2023 | 不透明类型，宏，所有权 |
| 6.0 | 预计 2025+ | 严格并发，编译改进 |

Swift 的演进体现了 Apple 对语言安全性和开发者体验的持续追求。从最初的「替代 Objective-C」到如今的「全栈系统编程语言」，Swift 已成长为具有广泛影响力的现代语言。
