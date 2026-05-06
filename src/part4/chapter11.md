# 第11章 错误处理的进化

Swift 的错误处理机制经历了持续的演进，从早期的 `NSError` 模式，到语言级别的 `throws`/`catch`，再到函数式的 `Result` 类型，以及 Swift 5.5 引入的异步错误处理。本章将系统性地梳理这些概念，帮助你在不同的场景下做出正确的选择。

## 11.1 空值 vs 错误 vs 崩溃：何时使用哪种

在 Swift 中，当一个操作失败时，我们通常有三种选择：返回 `nil`（空值）、抛出错误、或者触发崩溃。理解三者的适用场景是编写健壮代码的第一步。

### Optional（空值）的适用场景

当"没有值"是一个**预期之中的、正常的**可能性时，使用 Optional：

```swift
// 从字典中取值——键可能不存在，这是正常情况
let scores = ["Alice": 95, "Bob": 87]
let aliceScore = scores["Alice"]  // Int? = Optional(95)
let charlieScore = scores["Charlie"]  // Int? = nil

// 整数转换——输入可能不是合法数字
let number = Int("42")    // Optional(42)
let notNumber = Int("abc")  // nil

// 可选绑定处理
if let score = scores["Bob"] {
    print("Bob 的分数是 \(score)")
} else {
    print("没有找到 Bob 的成绩")
}
```

**适用场景**：
- 查找操作（字典取值、集合中查找元素）
- 类型转换（字符串转数字、JSON 解析中的可选字段）
- 获取可能不存在的对象属性

### throws（错误）的适用场景

当操作**可能失败，且失败的原因对调用者有意义**时，使用错误：

```swift
enum FileError: Error {
    case notFound(path: String)
    case insufficientPermissions
    case corruptedData
}

func readFile(at path: String) throws -> String {
    guard path.hasPrefix("/") else {
        throw FileError.notFound(path: path)
    }
    // ... 读取文件逻辑
    return "file content"
}

// 调用方需要明确处理失败的原因
do {
    let content = try readFile(at: "/etc/config")
    print(content)
} catch FileError.notFound(let path) {
    print("文件 \(path) 不存在")
} catch FileError.insufficientPermissions {
    print("没有权限访问")
} catch {
    print("未知错误：\(error)")
}
```

**适用场景**：
- 网络请求失败（连接超时、服务器错误）
- 文件操作（文件不存在、权限不足）
- 数据解析（格式错误、字段缺失）

### 崩溃的适用场景

崩溃（使用 `fatalError`、`precondition`、`assert`）只用于**编程错误——即 bug**，而不是运行时错误：

```swift
// ✅ 合理的崩溃：前置条件不满足说明代码有 bug
func divide(_ a: Int, by b: Int) -> Int {
    precondition(b != 0, "除数不能为 0，这是一个编程错误")
    return a / b
}

// ✅ 合理的崩溃：使用了未实现的必要方法
class Model {
    func save() {
        // 子类必须重写此方法
        fatalError("必须在子类中实现 save() 方法")
    }
}

// ❌ 不合理的崩溃：用崩溃处理用户输入
func processAge(_ input: String) -> Int {
    // 如果用户输入了 "abc"，完全不应该崩溃
    guard let age = Int(input) else {
        fatalError("不合法输入")  // 糟糕的设计！
    }
    return age
}
```

### 三者的选择矩阵

| 场景 | 使用方式 | 示例 |
|------|---------|------|
| 值不存在是预期行为 | `nil` | 字典取值 |
| 失败原因需要传递给调用者 | `throws` | 网络请求 |
| 程序状态错误，无法恢复 | `fatalError` | 数组越界 |
| 开发期调试 | `assert` | 调试期的不变量检查 |

## 11.2 throws 与 try

Swift 的 `throws` 关键字标记一个函数可能抛出错误，而 `try` 则用于调用这样的函数。这个设计让错误传播变得显式化。

### 声明抛出函数

```swift
enum ValidationError: Error {
    case emptyField(fieldName: String)
    case invalidFormat(fieldName: String, expected: String)
    case tooLong(fieldName: String, maxLength: Int)
}

struct UserRegistration {
    let username: String
    let email: String
    let password: String
    
    static func validate(username: String, email: String, password: String) throws -> UserRegistration {
        guard !username.trimmingCharacters(in: .whitespaces).isEmpty else {
            throw ValidationError.emptyField(fieldName: "用户名")
        }
        guard username.count >= 3 else {
            throw ValidationError.invalidFormat(fieldName: "用户名", expected: "至少 3 个字符")
        }
        guard email.contains("@") else {
            throw ValidationError.invalidFormat(fieldName: "邮箱", expected: "包含 @ 符号的合法邮箱")
        }
        guard password.count >= 8 else {
            throw ValidationError.tooLong(fieldName: "密码", maxLength: 8)
        }
        return UserRegistration(username: username, email: email, password: password)
    }
}
```

### 错误的自动传播

一个 `throws` 函数可以调用另一个 `throws` 函数，错误会自动向上传播：

```swift
func saveUser(username: String, email: String, password: String) throws {
    // validate 抛出的错误会自动传播到 saveUser 的调用方
    let user = try UserRegistration.validate(username: username, email: email, password: password)
    // ... 保存到数据库
}
```

### 非抛出函数不能调用抛出函数

```swift
// ❌ 编译错误：没有用 try 调用抛出函数
// func badProcess() {
//     let user = try UserRegistration.validate(username: "", email: "", password: "")
// }

// ✅ 必须用 throws 或 do-catch
func goodProcess() throws {
    let user = try UserRegistration.validate(username: "alice", email: "alice@example.com", password: "12345678")
}
```

## 11.3 do-catch 的精细捕获

`do-catch` 提供了对不同类型错误进行精细捕获的能力。Swift 的 catch 块支持模式匹配，这让我们可以根据错误的具体类型和关联值做不同的处理。

### 按类型捕获

```swift
do {
    let user = try UserRegistration.validate(username: "", email: "invalid", password: "short")
} catch ValidationError.emptyField(let field) {
    print("字段 \(field) 不能为空")
} catch ValidationError.invalidFormat(let field, let expected) {
    print("字段 \(field) 格式不正确，期望：\(expected)")
} catch ValidationError.tooLong(let field, let maxLength) {
    print("字段 \(field) 过长，最大允许 \(maxLength) 个字符")
} catch {
    // 兜底捕获——抓住了所有未匹配的错误
    print("未知验证错误：\(error)")
}
```

### 复合条件的 catch

catch 也可以结合 `where` 子句做更精细的匹配：

```swift
do {
    let data = try fetchData(from: url)
    try process(data)
} catch NetworkError.timeout where currentNetworkStatus == .poor {
    // 在网络状况差的情况下超时，提示用户检查网络
    showRetryPrompt(message: "网络状况不佳，是否重试？")
} catch NetworkError.timeout {
    // 其他超时情况
    showError(message: "请求超时，请稍后重试")
} catch NetworkError.serverError(let code) where code >= 500 {
    // 服务器端错误
    showError(message: "服务器错误 (\(code))，请稍后重试")
} catch let error as LocalizedError {
    // 所有实现了 LocalizedError 协议的错误
    showError(message: error.localizedDescription)
} catch {
    showError(message: "发生未知错误")
}
```

### 错误转换

有时我们需要将底层的错误转换为上层业务逻辑能够理解的错误类型：

```swift
enum ServiceError: Error {
    case networkFailure(underlying: Error)
    case authenticationFailed
    case rateLimited(retryAfter: Int)
}

func performPayment(amount: Double) throws -> Receipt {
    do {
        return try paymentService.charge(amount)
    } catch PaymentError.insufficientFunds {
        throw ServiceError.authenticationFailed
    } catch PaymentError.cardDeclined(let reason) {
        throw ServiceError.networkFailure(underlying: PaymentError.cardDeclined(reason: reason))
    } catch {
        throw ServiceError.networkFailure(underlying: error)
    }
}
```

## 11.4 Result 类型：错误建模的新思路

Swift 5.0 正式引入了标准库中的 `Result<Success, Failure>` 类型，其中 `Failure` 必须遵 `Error` 协议。`Result` 类型提供了不依赖 `throws`/`catch` 的错误处理方式，特别适合异步操作和函数式编程风格。

### Result 基础用法

```swift
func fetchUser(id: Int) -> Result<User, NetworkError> {
    guard id > 0 else {
        return .failure(.invalidParameter)
    }
    // 模拟网络请求
    if id == 1 {
        return .success(User(id: 1, name: "Alice"))
    } else {
        return .failure(.notFound)
    }
}

let result = fetchUser(id: 1)

// 使用 switch 处理
switch result {
case .success(let user):
    print("获取用户成功：\(user.name)")
case .failure(let error):
    print("获取失败：\(error)")
}

// 使用 Result 的方法链
let name = result
    .map { $0.name }
    .mapError { $0 as Error }
```

### Result 与 throws 的转换

`Result` 和 `throws` 可以互相转换：

```swift
// throws -> Result
func toResult<T>(_ closure: () throws -> T) -> Result<T, Error> {
    return Result(catching: closure)
}

let result = toResult {
    try UserRegistration.validate(username: "alice", email: "alice@example.com", password: "12345678")
}

// Result -> throws
func fetchConfig() throws -> Config {
    let result = loadConfigFromFile()
    return try result.get()  // 如果 result 是 .failure，get() 会抛出错误
}
```

### 组合多个 Result

`Result` 的真正威力在于组合多个可能失败的操作：

```swift
func validateAll(fields: [(String, String, String)]) -> Result<[UserRegistration], Error> {
    // 使用 map 收集所有注册结果
    let results = fields.map { (username, email, password) in
        Result { try UserRegistration.validate(username: username, email: email, password: password) }
    }
    
    // 将所有 Result 合并为一个：全部成功才返回成功
    return results.reduce(Result.success([])) { partialResult, nextResult in
        switch (partialResult, nextResult) {
        case (.success(var users), .success(let user)):
            users.append(user)
            return .success(users)
        case (.failure(let error), _):
            return .failure(error)
        case (_, .failure(let error)):
            return .failure(error)
        }
    }
}
```

### 使用 Result 包装异步回调

在 Swift Concurrency 普及之前，`Result` 是处理异步回调错误的标准方式：

```swift
typealias CompletionHandler = (Result<Data, NetworkError>) -> Void

func fetchData(from url: URL, completion: @escaping CompletionHandler) {
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(.failure(.transportError(error)))
            return
        }
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            completion(.failure(.serverError))
            return
        }
        guard let data = data else {
            completion(.failure(.noData))
            return
        }
        completion(.success(data))
    }.resume()
}
```

## 11.5 try? 与 try! 的选择

`try?` 和 `try!` 是两种特殊的错误处理方式，它们各有明确的适用场景。

### try?：将错误转换为可选值

`try?` 将可能抛出错误的表达式的结果封装为 Optional：成功时返回 `.some(value)`，失败时返回 `nil`：

```swift
// 传统方式
do {
    let config = try loadConfig()
    useConfig(config)
} catch {
    // 静默处理——使用默认配置
    useConfig(Config.default)
}

// 使用 try? —— 更简洁
if let config = try? loadConfig() {
    useConfig(config)
} else {
    useConfig(Config.default)
}
```

**try? 适用的场景**：
- 调用方**不关心具体的错误类型**，只关心成功或失败
- 失败时的默认行为是明确的
- 多个可能失败的操作组合使用

```swift
// 多个 try? 组合使用
let a = try? parseJSON(from: data1) // Data?
let b = try? parseJSON(from: data2) // Data?
let c = try? parseJSON(from: data3) // Data?

// 只需要至少一个成功
if let result = a ?? b ?? c {
    process(result)
}
```

### try!：强制解包错误

`try!` 断言操作**绝不会失败**，如果失败则会触发运行时崩溃：

```swift
// 当你确信操作不可能失败时
let image = try! loadEmbeddedResource(named: "logo.png")
// 如果你将 logo.png 打包在 app bundle 中，这个调用不可能失败

// 或者使用 fatalError 风格的显式说明
let config = try! Config(bundle: .main) // 内置配置如果失败，说明打包有问题
```

**try! 的适用场景**：
- 资源是应用内置的（打包在 Bundle 中）
- 硬编码的已知合法输入
- 测试代码中（测试失败应直接报错）

**⚠️ 注意**：`try!` 在产品代码中应该极少出现。过度使用 `try!` 意味着你把运行时错误转化为了崩溃，这违背了 Swift 的安全哲学。

### 三者的对比

| 关键字 | 成功时 | 失败时 | 使用频率 |
|--------|--------|--------|---------|
| `try` | 返回正常值 | 抛出错误给上层 | 频繁 |
| `try?` | 返回 Optional | 返回 nil | 适中 |
| `try!` | 返回正常值 | 触发崩溃 | 极少 |

## 11.6 错误处理与异步代码的结合

Swift 5.5 引入的 async/await 让异步代码的错误处理与同步代码保持了一致性——同样使用 `throws`/`catch`。

### async throws 函数

```swift
enum APIError: Error {
    case badRequest
    case unauthorized
    case notFound
    case rateLimited
    case serverError
}

func fetchUserProfile(userID: Int) async throws -> UserProfile {
    let url = URL(string: "https://api.example.com/users/\(userID)")!
    
    // async 网络请求
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard let httpResponse = response as? HTTPURLResponse else {
        throw APIError.serverError
    }
    
    switch httpResponse.statusCode {
    case 200:
        let decoder = JSONDecoder()
        return try decoder.decode(UserProfile.self, from: data)
    case 400:
        throw APIError.badRequest
    case 401:
        throw APIError.unauthorized
    case 404:
        throw APIError.notFound
    case 429:
        throw APIError.rateLimited
    default:
        throw APIError.serverError
    }
}
```

### 调用异步抛出函数

异步抛出函数的调用与同步抛出函数语法一致，只是多了 `await`：

```swift
func loadProfile() async {
    do {
        let profile = try await fetchUserProfile(userID: 42)
        updateUI(with: profile)
    } catch APIError.unauthorized {
        // 跳转到登录页面
        navigateToLogin()
    } catch APIError.notFound {
        showError("用户不存在")
    } catch {
        showError("加载失败：\(error.localizedDescription)")
    }
}
```

### 异步序列中的错误处理

`AsyncSequence` 同样支持错误处理：

```swift
func processEvents() async {
    do {
        for try await event in eventStream {
            handleEvent(event)
        }
    } catch {
        print("事件流中断：\(error)")
    }
}
```

### Task 与结构化并发的错误处理

```swift
func loadDashboard() async throws -> Dashboard {
    // 并发执行多个任务
    async let userProfile = fetchUserProfile(userID: 42)
    async let notifications = fetchNotifications()
    async let recommendations = fetchRecommendations()
    
    // 如果任意一个任务抛出错误，整个 async let 都会抛出
    // 这意味着如果三个请求中有一个失败，整个 dashboard 加载失败
    return try await Dashboard(
        profile: userProfile,
        notifications: notifications,
        recommendations: recommendations
    )
}
```

如果希望更精细地控制错误——比如某个子请求失败时使用默认值——可以用结构化任务组：

```swift
func loadDashboardResilient() async -> Dashboard {
    var profile: UserProfile?
    var notifications: [Notification] = []
    var recommendations: [Recommendation] = []
    
    // profile 失败时使用 nil 兜底
    profile = try? await fetchUserProfile(userID: 42)
    
    // 使用 TaskGroup 并发执行多个请求，独立处理错误
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            if let result = try? await fetchNotifications() {
                notifications = result
            }
        }
        group.addTask {
            if let result = try? await fetchRecommendations() {
                recommendations = result
            }
        }
    }
    
    return Dashboard(profile: profile, notifications: notifications, recommendations: recommendations)
}
```

### 异步错误处理的演进路径

回顾 Swift 错误处理的发展，我们可以清晰地看到一条演进路径：

1. **Swift 1-2**：`NSError` 指针模式（Objective-C 遗产）
2. **Swift 2-4**：`throws`/`catch` 和 `try?`/`try!` 引入，错误处理语言化
3. **Swift 5.0**：`Result` 类型加入标准库
4. **Swift 5.5+**：async/await 与 `throws` 无缝结合

## 小结

本章从选择空值、错误还是崩溃的决策框架出发，系统性地介绍了 Swift 的类型安全错误处理体系。无论你是使用传统的 `throws`/`do-catch`，还是函数式的 `Result` 类型，或者是在异步代码中处理错误，Swift 都提供了富有表现力和安全性的工具。

理解这些错误处理机制的适用场景和最佳实践，是写出健壮、可维护的 Swift 代码的关键。下一章，我们将转向函数式编程的世界，探索 Swift 如何支持函数式编程范式。
