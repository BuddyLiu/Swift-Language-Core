# 第20章 包管理与模块化

## 20.1 Swift Package Manager

Swift Package Manager（简称 SPM）是 Apple 官方提供的包管理工具，自 Swift 3.0 起集成在 Swift 编译工具链中。与 CocoaPods 或 Carthage 等第三方工具不同，SPM 与 Swift 编译器、构建系统深度集成，能在编译层面提供更精确的依赖解析和模块化支持。

### Package 的基本结构

一个 SPM 包的典型目录结构如下：

```
MyLibrary/
├── Package.swift          # 包清单，描述元数据、依赖和目标
├── Sources/
│   └── MyLibrary/
│       └── MyLibrary.swift
├── Tests/
│   └── MyLibraryTests/
│       └── MyLibraryTests.swift
└── README.md
```

`Package.swift` 是整个包的核心配置文件，使用 Swift 语言本身编写，这意味着它的语法、类型系统和工具链完全一致。

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyLibrary",
    platforms: [
        .iOS(.v16),
        .macOS(.v13)
    ],
    products: [
        .library(
            name: "MyLibrary",
            targets: ["MyLibrary"]
        ),
        .library(
            name: "MyLibraryDynamic",
            type: .dynamic,
            targets: ["MyLibrary"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.0.0"),
    ],
    targets: [
        .target(
            name: "MyLibrary",
            dependencies: [
                .product(name: "Logging", package: "swift-log"),
            ]
        ),
        .testTarget(
            name: "MyLibraryTests",
            dependencies: ["MyLibrary"]
        ),
    ]
)
```

### Package 的三种类型

SPM 将包划分为三种目标类型：

1. **`target`**：最常见的类型，编译为一个模块。一个 `.target` 会生成一个独立的模块，模块内部的所有 `public` 和 `open` 声明可以被外部访问。
2. **`testTarget`**：测试目标，依赖对应的主目标，仅在测试时参与编译。
3. **`executableTarget`**：生成可执行文件的目标，包含 `main.swift` 或 `@main` 标记的入口点。

### 依赖解析的工作流

SPM 使用语义化版本（Semantic Versioning）进行依赖解析。当你在 `Package.swift` 中声明依赖时：

```swift
.package(url: "https://github.com/apple/swift-algorithms.git", from: "1.0.0")
```

这意味着接受版本 `1.0.0` 到 `2.0.0`（不包含）之间的任何兼容版本。SPM 会通过 **依赖图解析** 算法，找出满足所有包约束条件的一组版本。

解析完成后，SPM 会生成 `Package.resolved` 文件，锁定所有依赖的具体版本，确保团队协作时使用一致的版本。

### Package 的访问控制模型

在 SPM 中，一个包中的不同目标（Target）是独立的编译模块。模块间的可见性由 Swift 的访问控制修饰符决定：

- `public` 和 `open` 的声明可以被其他模块访问
- `internal` 及其他更严格的修饰符，仅在当前模块内部可见

这意味着，一个库的作者必须显式选择哪些 API 暴露给调用者，其他内部实现细节默认被隐藏。

## 20.2 访问控制：open/public/internal/fileprivate/private

Swift 提供了五级访问控制，从最开放到最严格排列如下：

### open

`open` 是最开放的访问级别，它允许外部模块访问**并继承**类或**覆写**方法。`open` 专为类设计，在框架和库中用于标识那些设计为允许子类化的类。

```swift
// MyLibrary 模块
open class Shape {
    open func draw() {
        print("Drawing shape")
    }

    public func area() -> Double {
        return 0
    }
}

// YourApp 模块
class Circle: Shape {
    override func draw() {
        print("Drawing circle")
    }

    // 不可以覆写 area()，因为它只标记为 public
}
```

`open` 和 `public` 的关键区别在于：`open` 允许跨模块继承和覆写，`public` 只允许访问，不允许继承。

### public

`public` 允许外部模块访问声明，但不允许外部模块继承或覆写。这是对外暴露 API 的默认选择。

```swift
public struct APIResponse<T: Codable>: Codable {
    public let code: Int
    public let message: String
    public let data: T

    public init(code: Int, message: String, data: T) {
        self.code = code
        self.message = message
        self.data = data
    }
}
```

使用 `public` 时需要注意：结构体的默认逐一构造器（Memberwise Initializer）是 `internal` 的，需要显式提供 `public init`。

### internal

`internal` 是默认访问级别。声明的实体只在当前模块内部可见。如果不写任何修饰符，编译器默认使用 `internal`。

```swift
// 只在模块内部使用的辅助类型
struct DatabasePool {
    let connections: [Connection]

    func acquire() -> Connection? {
        return connections.first
    }
}
```

### fileprivate

`fileprivate` 将访问范围限制在当前文件内。同一个文件中的多个类型、扩展可以互相访问 `fileprivate` 成员。

```swift
class AccountManager {
    fileprivate var accounts: [Account] = []

    func add(_ account: Account) {
        accounts.append(account)
    }
}

// 同一个文件中的扩展可以访问 fileprivate 属性
extension AccountManager {
    func totalBalance() -> Double {
        accounts.reduce(0) { $0 + $1.balance }
    }
}
```

`fileprivate` 常用于将类型的一部分实现逻辑放在同文件的扩展中，同时避免暴露给文件外部。

### private

`private` 是最严格的访问级别，访问范围限制在声明所在的**花括号**（作用域）内。同一类型的不同扩展不能互相访问 `private` 成员。

```swift
struct Temperature {
    private var celsius: Double

    var fahrenheit: Double {
        celsius * 9 / 5 + 32
    }

    init(celsius: Double) {
        self.celsius = celsius
    }
}

// 编译错误：celsius 是 private 的
// extension Temperature {
//     mutating func reset() {
//         celsius = 0
//     }
// }
```

### 访问控制的一般原则

在实际项目中，遵循以下原则可以帮助你设计出清晰、安全的模块边界：

1. **最小暴露原则**：默认使用 `internal`，只有当某个 API 需要被其他模块使用时才提升为 `public` 或 `open`。
2. **协议与公开 API**：如果协议是 `public` 的，其关联类型和方法也必须保持 `public`。协议的 `public` 一致性声明也必须是 `public` 的。
3. **内部辅助类型**：使用 `fileprivate` 或嵌套类型的 `private` 来隐藏实现细节。
4. **单元测试访问**：对于测试目标，可以使用 `@testable import MyModule` 来访问 `internal` 级别的声明——但这是在测试环境中的特例，不应在生产代码中使用。

```swift
// 测试代码
@testable import MyLibrary
// 现在可以访问 MyLibrary 中所有 internal 声明
```

## 20.3 协议导向架构

在模块化设计过程中，协议起到了关键的**解耦**作用。协议导向架构（Protocol-Oriented Architecture）是本书第 4 章所介绍的面向协议编程在工程层面的具体落地。

### 模块间依赖的面向协议重构

假设我们的应用包含三个模块：`Domain`、`Networking` 和 `UI`。在传统的依赖关系中，`UI` 模块会直接依赖 `Networking` 模块：

```
UI → Networking → Domain
```

这种紧耦合导致 `UI` 模块难以测试，也难以替换 `Networking` 的实现。通过引入协议，我们可以反转依赖方向：

```
UI → Domain ← Networking
```

`Domain` 模块定义服务协议，`Networking` 提供具体实现，`UI` 依赖于协议而非具体实现。

```swift
// Domain 模块
public protocol UserRepository {
    func fetchUser(id: String) async throws -> User
}

public struct User: Codable, Sendable {
    public let id: String
    public let name: String
}

// Networking 模块
import Domain

public struct RemoteUserRepository: UserRepository {
    let session: URLSession

    public init(session: URLSession = .shared) {
        self.session = session
    }

    public func fetchUser(id: String) async throws -> User {
        let (data, _) = try await session.data(from: URL(string: "https://api.example.com/users/\(id)")!)
        return try JSONDecoder().decode(User.self, from: data)
    }
}

// UI 模块
import Domain

struct UserView: View {
    let repository: UserRepository

    var body: some View {
        // 使用 repository 获取数据
    }
}
```

### 模块化网关

跨模块的协议设计需要特别注意版本兼容性和演进策略。一个常见的模式是使用**网关接口**——为每个跨模块交互定义一个专门的协议。

```swift
// Core 模块：定义网关协议
public protocol AnalyticsGateway: AnyObject {
    func track(event: String, parameters: [String: Any])
    func setUserID(_ id: String?)
}

// Analytics 模块：提供实现
import Core

public final class FirebaseAnalyticsGateway: AnalyticsGateway {
    public func track(event: String, parameters: [String: Any]) {
        // Firebase 的具体实现
    }

    public func setUserID(_ id: String?) {
        // Firebase 的具体实现
    }
}

// 业务模块：只依赖协议
import Core

struct CheckoutService {
    let analytics: AnalyticsGateway

    func completePurchase() {
        analytics.track(event: "purchase_completed", parameters: [:])
    }
}
```

## 20.4 模块间的依赖反转

依赖反转原则（Dependency Inversion Principle，DIP）是 SOLID 原则中的关键一条，其核心思想是：**高层模块不应依赖低层模块，二者都应依赖抽象**。在 Swift 的模块化架构中，这一原则通过协议得以自然实现。

### 传统依赖 vs 反转后

假设一个订单处理系统：

**传统写法（直接依赖）：**

```swift
import StripeKit

public class OrderProcessor {
    private let paymentGateway = StripePaymentGateway()

    public func process(order: Order) {
        paymentGateway.charge(amount: order.total)
    }
}
```

这种写法有多个问题：`OrderProcessor` 直接依赖于 `StripePaymentGateway`，如果要切换到 Square 或者 Apple Pay，必须修改 `OrderProcessor` 的源码；同时，单元测试也难以模拟支付网关。

**反转后的写法：**

```swift
// 支付模块 —— 定义协议
public protocol PaymentGateway {
    func charge(amount: Decimal, currency: String) async throws -> TransactionResult
}

// 业务模块 —— 依赖于抽象
public class OrderProcessor {
    private let paymentGateway: PaymentGateway

    public init(paymentGateway: PaymentGateway) {
        self.paymentGateway = paymentGateway
    }

    public func process(order: Order) async throws {
        let result = try await paymentGateway.charge(
            amount: order.total,
            currency: order.currency
        )
        // 处理结果
    }
}

// Stripe 模块 —— 提供实现
import Payment

public struct StripeGateway: PaymentGateway {
    let apiKey: String

    public func charge(amount: Decimal, currency: String) async throws -> TransactionResult {
        // Stripe API 调用
    }
}

// 测试模拟
import Payment
import XCTest

class MockPaymentGateway: PaymentGateway {
    var chargeCallCount = 0
    var shouldSucceed = true

    func charge(amount: Decimal, currency: String) async throws -> TransactionResult {
        chargeCallCount += 1
        if shouldSucceed {
            return TransactionResult(success: true, transactionID: "mock-123")
        } else {
            throw PaymentError.insufficientFunds
        }
    }
}
```

### 依赖注入容器

在较大规模的 Swift 项目中，通常使用依赖注入容器来管理模块间的依赖关系。

```swift
public final class DIContainer {
    private var factories: [ObjectIdentifier: () -> Any] = [:]

    public func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        factories[ObjectIdentifier(type)] = factory
    }

    public func resolve<T>(_ type: T.Type) -> T {
        guard let factory = factories[ObjectIdentifier(type)],
              let instance = factory() as? T else {
            fatalError("No registered factory for \(T.self)")
        }
        return instance
    }
}

// 使用
let container = DIContainer()
container.register(PaymentGateway.self) { StripeGateway(apiKey: "sk_test_...") }
container.register(OrderProcessor.self) {
    OrderProcessor(paymentGateway: container.resolve(PaymentGateway.self))
}
```

### 模块化组织的反模式

最后，列出几种常见的反模式，值得我们在模块化设计中警惕：

1. **循环依赖**：模块 A 依赖模块 B，模块 B 又依赖模块 A。这通常意味着模块拆分不合理，可以通过提取公共抽象来打破循环。
2. **万能模块**：将所有代码放在一个巨大的模块中。这不仅增加了编译时间，还模糊了模块边界，使代码难以维护。
3. **过度拆分**：将每个类都做成独立模块。这会导致模块数量爆炸，增加构建和管理的复杂度。
4. **裸协议公开**：在公开协议中暴露了太多方法，导致实现者负担过重。应保持协议的**最小化**原则（Interface Segregation）。

## 小结

本章我们学习了 Swift 包管理与模块化的核心知识：SPM 的基本概念和使用方法，五级访问控制修饰符的精确含义与适用场景，如何利用协议设计松耦合的模块架构，以及依赖反转原则在模块化设计中的具体实践。这些知识是构建大型 Swift 应用的基石，无论你开发的是 iOS 应用、macOS 应用还是服务器端 Swift 项目，合理的模块化设计都是保证代码长期可维护性的关键。
