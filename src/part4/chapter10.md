# 第10章 模式匹配与条件逻辑

模式匹配是 Swift 语言中最具表现力的特性之一。它不仅存在于 `switch` 语句中，还渗透到了 `if`、`guard`、`for` 乃至赋值操作的方方面面。理解模式匹配，意味着你能够写出更简洁、更安全、更具可读性的 Swift 代码。

## 10.1 switch 的完备性检查：必须覆盖所有分支

Swift 的 `switch` 语句与 C 语言家族最大的不同在于：**它必须覆盖所有可能的情况**。这一设计并非为了增加开发者的负担，而是从根本上消除了"遗漏分支"这一常见的运行时错误来源。

### 枚举的穷举保证

对于 Swift 的枚举类型，编译器会强制要求 `switch` 覆盖每一个 case：

```swift
enum NetworkState {
    case connecting
    case connected
    case disconnected
}

// ✅ 编译通过：覆盖了所有分支
func handleState(_ state: NetworkState) {
    switch state {
    case .connecting:
        print("连接中...")
    case .connected:
        print("已连接")
    case .disconnected:
        print("未连接")
    }
}

// ❌ 编译错误：Switch must be exhaustive
// func handleState(_ state: NetworkState) {
//     switch state {
//     case .connecting:
//         print("连接中...")
//     }
// }
```

### default 与 unknown 的权衡

可以使用 `default` 来匹配所有未显式列出的分支，但这会丧失编译器的穷举检查能力：

```swift
enum HTTPStatusCode: Int {
    case ok = 200
    case created = 201
    case notFound = 404
    case serverError = 500
}

func describeCode(_ code: HTTPStatusCode) -> String {
    switch code {
    case .ok:
        return "请求成功"
    case .notFound:
        return "资源未找到"
    default:
        // 编译器不再检查是否遗漏了 .created 和 .serverError
        return "其他状态码"
    }
}
```

当枚举新增 case 时，`default` 会静默地覆盖新分支，这可能导致逻辑遗漏。因此，团队的最佳实践是：**如果枚举的 case 是有限的、可穷举的，尽量显式写出所有分支**；仅在确实需要兜底逻辑时使用 `default`。

### 不可穷举类型的处理

对于 `Int`、`String` 等非穷举类型，必须包含 `default` 分支：

```swift
func describeNumber(_ x: Int) -> String {
    switch x {
    case 0:
        return "零"
    case 1...9:
        return "个位数"
    default:
        return "其他数值"
    }
}
```

## 10.2 值绑定与 where 子句：let、where 的灵活应用

值绑定（Value Binding）和 `where` 子句是模式匹配的两大利器，它们让 `switch` 的分支逻辑变得极其灵活。

### 值绑定 (let/var)

值绑定允许我们在匹配的同时，将匹配到的值绑定到变量或常量中：

```swift
enum PaymentResult {
    case success(transactionID: String, amount: Double)
    case failure(errorMessage: String)
    case pending(approvalCode: String)
}

func processPayment(_ result: PaymentResult) {
    switch result {
    case .success(let id, let amount):
        print("交易 \(id) 成功，金额：¥\(amount)")
        // 在分支内可以使用 id 和 amount
    case .failure(let message):
        print("支付失败：\(message)")
    case .pending(let code):
        print("等待审批：\(code)")
    }
}
```

也可以将 `let` 写在 case 的冒号前面，提取关联值：

```swift
// 更简洁的写法
switch result {
case let .success(id, amount):
    print("交易 \(id) 成功，金额：¥\(amount)")
case let .failure(message):
    print("支付失败：\(message)")
case let .pending(code):
    print("等待审批：\(code)")
}
```

### where 子句的条件增强

`where` 子句为分支添加额外的条件判断，它相当于在模式匹配的基础上叠加了一层逻辑过滤：

```swift
func describeTemperature(_ temp: Int) -> String {
    switch temp {
    case let t where t < -10:
        return "严寒，温度 \(t)°C，注意保暖"
    case let t where t < 0:
        return "寒冷，温度 \(t)°C"
    case let t where t < 20:
        return "凉爽，温度 \(t)°C"
    case let t where t < 35:
        return "温暖，温度 \(t)°C"
    case let t where t >= 35:
        return "炎热，温度 \(t)°C，小心中暑"
    default:
        return "未知温度" // 理论上不会执行，但编译器要求
    }
}
```

`where` 还可以结合多个条件，组合出复杂的匹配规则：

```swift
enum UserAction {
    case login(username: String, isAdmin: Bool)
    case viewPage(pageID: Int)
    case logout
}

func handleAction(_ action: UserAction) {
    switch action {
    case .login(let username, true) where username.hasPrefix("admin_"):
        print("超级管理员 \(username) 登录")
    case .login(let username, true):
        print("管理员 \(username) 登录")
    case .login(let username, false):
        print("普通用户 \(username) 登录")
    case .viewPage(let id) where id > 0 && id <= 100:
        print("查看公开页面 #\(id)")
    case .viewPage:
        print("查看受限页面")
    case .logout:
        print("登出")
    }
}
```

## 10.3 if case、guard case、for case

`switch` 虽然强大，但在只需要匹配单个模式时显得过于冗长。Swift 提供了 `if case`、`guard case` 和 `for case` 三种语法糖，让我们在特定场景下可以更简洁地进行模式匹配。

### if case：单分支模式匹配

当只关心一个分支时，`if case` 可以让代码更加紧凑：

```swift
enum OptionalValue<T> {
    case some(T)
    case none
}

let result = OptionalValue.some(42)

// switch 写法
switch result {
case .some(let value):
    print("值为：\(value)")
default:
    break
}

// if case 写法 —— 更简洁
if case let .some(value) = result {
    print("值为：\(value)")
}
```

`if case` 也支持 `where` 子句：

```swift
let networkState: NetworkState = .connected

if case .connected = networkState {
    print("已连接到网络")
}

// 结合 where 子句
enum OptionalInt {
    case some(Int)
    case none
}

let number = OptionalInt.some(42)
if case let .some(x) = number, x > 0 {
    print("\(x) 是正数")
}
```

### guard case：提前退出模式

当模式不匹配时需要提前退出函数，`guard case` 是理想选择：

```swift
func processOptional(_ value: OptionalInt) {
    guard case let .some(x) = value else {
        print("值为空，无法处理")
        return
    }
    // 这里 x 已在作用域内可用
    print("处理数值：\(x * 2)")
}
```

### for case：遍历中的模式过滤

当遍历一个集合时，如果你只关心其中符合特定模式的元素，`for case` 可以避免手写 `if` 判断：

```swift
let mixedValues: [Any] = [1, "hello", 3.14, true, 42, "world"]

// 只处理字符串类型的元素
for case let str as String in mixedValues {
    print("字符串：\(str)")
}
// 输出：字符串：hello / 字符串：world

// 只处理偶数
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
for case let n in numbers where n.isMultiple(of: 2) {
    print(n, terminator: " ")
}
// 输出：2 4 6 8 10
```

配合 Optional 模式，`for case let` 可以优雅地过滤非 nil 值：

```swift
let optionalNumbers: [Int?] = [1, nil, 3, nil, 5, 6]

// 只处理非 nil 的值
for case let .some(n) in optionalNumbers {
    print(n, terminator: " ")
}
// 输出：1 3 5 6

// 更简洁的等价写法
for case let n? in optionalNumbers {
    print(n, terminator: " ")
}
// 输出：1 3 5 6
```

## 10.4 通配符与元组匹配

### 通配符 _

下划线 `_` 作为通配符（wildcard pattern），可以匹配任何值但不绑定：

```swift
let someString: String? = "Hello"

// 只关心是否为 nil，不关心具体值
if case .some(_) = someString {
    print("字符串不为空")
}

// 更优雅的写法
if someString != nil {
    print("字符串不为空")
}
```

在 switch 中，通配符常用于忽略不需要的关联值：

```swift
enum ServerResponse {
    case success(data: Data, statusCode: Int)
    case failure(error: Error, statusCode: Int)
}

func handleResponse(_ response: ServerResponse) {
    switch response {
    case .success(_, let code):
        print("请求成功，状态码：\(code)")
    case .failure(_, let code):
        print("请求失败，状态码：\(code)")
    }
}
```

### 元组模式匹配

元组模式（Tuple Pattern）是将多个值组合在一起进行匹配，这在处理多条件逻辑时极其强大：

```swift
let point = (x: 3, y: -2)

switch point {
case (0, 0):
    print("原点")
case (let x, 0):
    print("X 轴上的点，x=\(x)")
case (0, let y):
    print("Y 轴上的点，y=\(y)")
case (let x, let y) where x == y:
    print("对角线上的点，x=y=\(x)")
case (let x, let y) where x == -y:
    print("反对角线上的点")
case (let x, let y):
    print("普通点：\(x), \(y)")
}
```

元组匹配在业务逻辑中同样实用。比如，我们需要根据用户的状态和权限做决策：

```swift
enum Membership { case normal, vip, svip }
enum ContentType { case free, paid, exclusive }

func canAccess(content: ContentType, membership: Membership) -> Bool {
    switch (content, membership) {
    case (.free, _):       // 免费内容所有人都能看
        return true
    case (.paid, .normal): // 付费内容需要 VIP
        return false
    case (.paid, _):       // VIP 和 SVIP 可以看付费内容
        return true
    case (.exclusive, .svip): // 专属内容仅 SVIP
        return true
    case (.exclusive, _):
        return false
    }
}
```

多个值的区间匹配也可以借助元组：

```swift
func classify(score: Int, attendance: Int) -> String {
    switch (score, attendance) {
    case (90...100, 90...100):
        return "优秀：成绩优异且全勤"
    case (60...100, 0..<60):
        return "警告：虽然及格但出勤不足"
    case (0..<60, _):
        return "不及格"
    case (_, 0..<60):
        return "出勤不足"
    default:
        return "继续努力"
    }
}
```

## 10.5 自定义类型的模式匹配：~= 运算符重载

Swift 的模式匹配机制是可扩展的。`switch` 的每个 `case` 本质上是在调用 `~=` 运算符，它的签名是：

```swift
func ~= (pattern: Value, value: Value) -> Bool
```

通过重载 `~=`，我们可以让任何自定义类型参与模式匹配。

### 为自定义类型添加模式匹配支持

假设我们有一个 `IPAddress` 类型：

```swift
struct IPAddress: Equatable {
    let octets: [UInt8]  // 四个 0-255 的数组
    
    init?(_ string: String) {
        let parts = string.split(separator: ".").compactMap { UInt8($0) }
        guard parts.count == 4 else { return nil }
        octets = parts
    }
}

enum IPCategory {
    case localhost
    case privateNetwork
    case publicAddress
}

// 重载 ~= 运算符，使 IPAddress 可以在 switch 中匹配
func ~= (pattern: IPCategory, value: IPAddress) -> Bool {
    switch pattern {
    case .localhost:
        return value.octets == [127, 0, 0, 1]
    case .privateNetwork:
        // 10.x.x.x 或 192.168.x.x 或 172.16-31.x.x
        return value.octets[0] == 10
            || value.octets[0] == 192 && value.octets[1] == 168
            || value.octets[0] == 172 && (16...31).contains(value.octets[1])
    case .publicAddress:
        return true // 其余都是公网地址
    }
}

// 使用
let ip = IPAddress("192.168.1.1")!
switch ip {
case .localhost:
    print("本地回环地址")
case .privateNetwork:
    print("内网地址")
case .publicAddress:
    print("公网地址")
}
// 输出：内网地址
```

### 实现区间匹配

我们可以让自定义类型支持区间模式匹配：

```swift
struct Temperature: Comparable {
    let celsius: Double
    
    static func < (lhs: Temperature, rhs: Temperature) -> Bool {
        return lhs.celsius < rhs.celsius
    }
    
    static func == (lhs: Temperature, rhs: Temperature) -> Bool {
        return lhs.celsius == rhs.celsius
    }
}

// 让 Temperature 可以匹配 Range
func ~= (pattern: Range<Temperature>, value: Temperature) -> Bool {
    return pattern.contains(value)
}

let temp = Temperature(celsius: 25)
switch temp {
case Temperature(celsius: -100)..<Temperature(celsius: 0):
    print("冰点以下")
case Temperature(celsius: 0)..<Temperature(celsius: 20):
    print("凉爽")
case Temperature(celsius: 20)..<Temperature(celsius: 35):
    print("温暖")
default:
    print("炎热")
}
// 输出：温暖
```

### 更复杂的匹配逻辑

`~=` 还可以实现更高级的匹配场景，例如正则表达式匹配：

```swift
struct Regex {
    let pattern: String
    
    func match(_ input: String) -> Bool {
        return input.range(of: pattern, options: .regularExpression) != nil
    }
}

func ~= (pattern: Regex, value: String) -> Bool {
    return pattern.match(value)
}

let emailRegex = Regex(pattern: "^[A-Z0-9._%+-]+@[A-Z0-9.-]+\\.[A-Z]{2,}$")
let phoneRegex = Regex(pattern: "^1[3-9]\\d{9}$")

func validateInput(_ input: String) {
    switch input {
    case emailRegex:
        print("有效的邮箱地址")
    case phoneRegex:
        print("有效的手机号")
    default:
        print("不符合已知格式")
    }
}

validateInput("user@example.com")    // 有效的邮箱地址
validateInput("13800138000")         // 有效的手机号
validateInput("hello world")         // 不符合已知格式
```

## 小结

本章介绍了 Swift 模式匹配的完整体系。从 `switch` 的穷举性保证，到值绑定和 `where` 子句的灵活应用，再到 `if case`、`guard case` 和 `for case` 的语法糖，以及通配符与元组匹配的优雅组合，最后到 `~=` 运算符重载带来的无限扩展性，这些特性共同构成了 Swift 强大的模式匹配武器库。

掌握模式匹配不仅能让你写出更安全的代码——因为编译器会替你检查遗漏的分支——还能让代码更加简洁和表达力更强。下一章，我们将探讨 Swift 中另一个重要的语言特性：错误处理的进化。
