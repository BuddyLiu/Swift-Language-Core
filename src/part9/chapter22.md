# 第22章 设计决策指南：何时用何物

Swift 是一门多范式的语言，它提供了多种解决同一问题的手段。这种丰富性既是优势，也带来选择困难——当结构体和类都能实现某个功能时，当协议继承和泛型约束可以互相替代时，我们该依据什么做出决策？

本章将逐一分析这些核心设计决策，提供清晰的判断标准和实践经验。

## 22.1 值类型还是类

这是 Swift 开发者面临的最基本的选择。结构体（值类型）和类（引用类型）各有其设计目标和适用场景。

### 优先选择值类型

Swift 标准库的设计遵循"值类型优先"的原则：`String`、`Array`、`Dictionary`、`Int`、`Double` 等全是结构体。这并非巧合，而是深思熟虑的设计决策。

**选择值类型的理由：**

1. **数据独立性**：值类型总是独立副本，传递后不会被其他代码意外修改。
2. **线程安全**：值类型没有共享状态风险，天然适用于并发环境。
3. **无引用循环**：值类型不参与 ARC，不存在循环引用问题。
4. **编译器优化**：小而不可变的值类型可以被编译器深度优化，内联到寄存器中。

```swift
// 值类型的自然选择
struct Point {
    var x: Double
    var y: Double
}

// 修改副本不影响原始值
var p1 = Point(x: 0, y: 0)
var p2 = p1
p2.x = 10
print(p1.x) // 0 —— 完全独立
```

### 引用类型的选择场景

在以下场景中，类是更好的选择：

1. **需要共享可变状态**：例如应用的全局 Session 或缓存。
2. **需要唯一标识**：实体对象（如数据库中的用户记录）有身份概念，即便属性完全相同的两个对象也应视为不同实例。
3. **需要与 Objective-C 互操作**：某些 OC 运行时特性（如 KVO）只能用于类。
4. **需要继承体系**：虽然 Swift 推荐用协议替代继承，但某些场景（如抽象工厂）仍需要类继承。
5. **生命周期管理**：需要析构器 `deinit` 来执行清理操作。

```swift
// 引用类型的适用场景：具有唯一身份的可变实体
class UserSession {
    let userID: String
    var authToken: String
    var lastActive: Date

    init(userID: String, authToken: String) {
        self.userID = userID
        self.authToken = authToken
        self.lastActive = Date()
    }

    deinit {
        // 清理会话缓存
        SessionManager.shared.removeSession(userID: userID)
    }
}

// 两个 UserSession，即便属性完全相同，身份也不同
let session1 = UserSession(userID: "1", authToken: "token1")
let session2 = UserSession(userID: "1", authToken: "token1")
print(session1 === session2) // false —— 不同实例
```

### 决策流程图

```
这个数据需要唯一身份吗？
├── 是 → 类（引用类型）
└── 否 → 这个数据会被共享并多处修改吗？
    ├── 是 → 类，或使用引用语义的包装类型
    └── 否 → 结构体（值类型）
        ├── 这个结构体很大吗（> 3-4 个字长）？
        │   ├── 是 → 考虑写时复制（Copy-on-Write）
        │   └── 否 → 结构体，编译器会高效内联
        └── 需要析构器来释放非内存资源吗？
            └── 是 → 类（引用类型）
```

### 结构体的"大"问题

一个常见的警告是"结构体太大时要改用类"。实际上，这个建议需要更精确地理解：

结构体作为值类型，在作为函数参数或返回值时会被复制。对于大于 3-4 个字长（约 24-32 字节）的结构体，复制成本会超过引用计数的成本。此时可以使用**写时复制**模式：

```swift
// 写时复制：用引用类型做存储，值类型做接口
final class _Storage {
    var items: [Int] = []
}

struct Container {
    private var _storage = _Storage()

    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&_storage) {
            _storage = _Storage()
            _storage.items = _storage.items
        }
    }

    mutating func append(_ item: Int) {
        ensureUnique()
        _storage.items.append(item)
    }
}
```

不过，Swift 标准库中的 `Array`、`String`、`Dictionary` 等已内置了写时复制，无需自定义。

## 22.2 继承还是协议

在传统 OOP 语言中，继承是代码复用的主要手段。Swift 虽然保留了类继承，但提供了更强大的协议系统作为替代。

### 用协议替代继承的理由

**1. 值类型支持**：结构体和枚举不能继承，但可以遵守协议。使用协议可以让值类型也具备多态能力。

```swift
// 继承方案 —— 只适用于类
class Animal {
    func makeSound() -> String { "" }
}
class Dog: Animal {
    override func makeSound() -> String { "Woof" }  // 类需要 override

}

// 协议方案 —— 值类型和引用类型都适用
protocol SoundMakable {
    func makeSound() -> String
}
struct Dog: SoundMakable {
    func makeSound() -> String { "Woof" }  // 不需要 override
}
```

**2. 松耦合**：继承建立了一种"is-a"的强耦合关系，子类与父类紧密绑定。协议描述的是"can-do"能力，实现者之间没有继承关系。

**3. 多重能力**：Swift 不允许多重继承，但类型可以遵守多个协议。这使得建模更灵活。

```swift
protocol Flyable { func fly() }
protocol Swimmable { func swim() }

// 鸭子既能飞又能游，不需要复杂的继承层级
struct Duck: Flyable, Swimmable {
    func fly() { print("飞") }
    func swim() { print("游") }
}
```

### 合理使用继承的场景

虽然协议更好，但继承在某些场景下仍然合理：

1. **共享可变状态和逻辑**：多个子类需要共享父类的存储属性和方法实现。
2. **模板方法模式**：父类定义骨架算法，子类覆写特定步骤。
3. **需要 Objective-C 兼容性**：某些 Cocoa/ObjC 框架要求子类化（如 `UIViewController`）。

```swift
class AbstractDataValidator {
    private var errors: [String] = []  // 共享可变状态

    func validate() -> Bool {
        errors.removeAll()
        performValidation()
        return errors.isEmpty
    }

    func performValidation() {
        // 子类覆写
    }

    func addError(_ message: String) {
        errors.append(message)
    }
}

final class EmailValidator: AbstractDataValidator {
    override func performValidation() {
        // 具体验证逻辑
    }
}
```

### 决策指南

```
需要定义一组能力，且可能被不相关的类型实现？
├── 是 → 使用协议
└── 否 → 需要共享带状态的实现？
    ├── 是 → 使用类继承或协议扩展（优先协议扩展）
    └── 否 → 使用协议 + 默认实现
```

## 22.3 throws 还是 Result

Swift 提供了两种错误处理机制：`throws`（带 `try` 的自动传播）和 `Result`（显式的成功/失败枚举）。两者并非非此即彼，而是服务于不同的使用场景。

### 使用 throws 的场景

`throws` 适用于**错误需要自动传播**的场景——函数内部遇到的错误，调用方可能不关心细节，只想让错误继续向上传播。

```swift
func loadConfig() throws -> Config {
    let data = try FileManager.default.contents(atPath: "/app/config.json")
    return try JSONDecoder().decode(Config.self, from: data)
}

func startApp() throws {
    let config = try loadConfig()
    // 使用 config
}
```

`throws` 的优点：
- 语法简洁，不需要每一层都手动处理错误
- 与异步函数配合良好：`async throws`
- 调用方只需在适当层级捕获错误

使用 `throws` 的建议：
- 同步操作或异步操作中的临时性错误
- 调用方通常不需要区分多个错误类型
- 错误应被传播到某个集中处理点

### 使用 Result 的场景

`Result` 适用于**调用方必须显式处理成功/失败**的场景，尤其适合异步回调或需要对失败进行后续处理的逻辑。

```swift
func processPayment(amount: Decimal) -> Result<Transaction, PaymentError> {
    guard amount > 0 else {
        return .failure(.invalidAmount)
    }
    // 处理支付逻辑
    return .success(Transaction(id: "tx-123", amount: amount))
}

// 调用方必须处理两种结果
let result = processPayment(amount: 100)
switch result {
case .success(let tx):
    print("支付成功：\(tx.id)")
case .failure(let error):
    print("支付失败：\(error)")
    // 可以进行补偿操作
    if error == .invalidAmount {
        // 提示用户输入有效金额
    }
}
```

`Result` 的优点：
- 显式性：调用方无法忽略错误
- 可组合性：`map`、`flatMap` 等操作可以链式处理
- 错误不传播：错误不会越过当前作用域

### 混合使用

在实际项目中，常见的最佳实践是：内部函数使用 `throws` 简化传播路径，在 API 边界处转换为 `Result` 供调用方处理。

```swift
// 内部实现：使用 throws 简化传播
private func fetchRawData() throws -> Data { ... }
private func parseData(_ data: Data) throws -> Model { ... }

// 公开 API：转换为 Result
func loadModel() async -> Result<Model, AppError> {
    do {
        let data = try fetchRawData()
        let model = try parseData(data)
        return .success(model)
    } catch let error as NetworkError {
        return .failure(.network(error))
    } catch let error as DecodingError {
        return .failure(.decoding(error))
    } catch {
        return .failure(.unknown(error))
    }
}
```

### 决策指南

```
错误是否需要传播到上层调用方？
├── 是 → 用 throws
└── 否 → 调用方需要本地处理所有错误分支吗？
    ├── 是 → 用 Result
    └── 否 → 结合使用：throws 内部，Result 边界
```

## 22.4 同步还是异步

Swift 5.5 引入结构化并发后，同步和异步之间的选择变得更加明确。

### 优先使用同步

如果操作的耗时可以忽略不计（微秒级别），或者数据已经存在于内存中，应优先使用同步函数。同步函数的优势在于：

- 执行路径简单，易于理解和调试
- 没有线程切换开销
- 不需要 `await` 上下文，调用更灵活

```swift
// 同步操作：内存计算
func calculatePortfolioValue(holdings: [Holding]) -> Decimal {
    holdings.reduce(0) { $0 + ($1.currentValue ?? 0) }
}
```

### 何时使用异步

以下场景应使用 `async` 函数：

1. **I/O 操作**：网络请求、文件读写、数据库查询
2. **时间相关操作**：定时器、延迟、超时
3. **大量计算**：CPU 密集型任务，需要转移到后台队列
4. **跨系统调用**：访问系统服务或其他进程

```swift
// 异步操作：I/O 和网络
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}
```

### 不要混用的场景

同步和异步的混用可能导致死锁或线程爆炸。应避免：

- 在同步函数中调用异步函数（使用 `DispatchGroup` 或 `semaphore` 阻塞等待）
- 在 actor 的同步方法中执行耗时操作（会阻塞 actor 的执行器）

```swift
// 反模式：在同步函数中阻塞等待异步结果
func loadUserSync(id: String) -> User? {
    let semaphore = DispatchSemaphore(value: 0)
    var result: User?
    Task {
        result = try? await fetchUser(id: id)
        semaphore.signal()
    }
    semaphore.wait()  // 可能导致线程死锁
    return result
}
```

### 决策指南

```
操作会立即返回结果吗？
├── 是 → 同步函数
└── 否 → 操作的主要成本是什么？
    ├── I/O 或等待 → async 函数
    └── CPU 计算 → 考虑 async 函数中的 Task.detached
```

## 22.5 协议与泛型间权衡

协议和泛型都可以实现多态，但它们的机制和适用场景不同。

### 协议多态（动态派发）

协议使用存在类型（existential types）实现动态派发：

```swift
protocol Drawable {
    func draw()
}

// 存在类型：运行时动态派发
func render(_ objects: [Drawable]) {
    for object in objects {
        object.draw()  // 运行时确定具体类型
    }
}
```

优点：
- 可以放置不同类型的对象在同一个集合中
- 运行时动态，适用于插件架构和依赖注入

代价：
- 间接调用（虚函数表）开销
- 存在类型有额外的内存开销（存在容器）
- 某些协议（带 Self 或关联类型）不能作为存在类型使用

### 泛型多态（静态派发）

泛型通过类型参数实现静态多态：

```swift
// 泛型函数：编译时确定具体类型
func render<T: Drawable>(_ objects: [T]) where T: Equatable {
    for object in objects {
        object.draw()  // 编译时特化，直接调用
    }
}
```

优点：
- 编译时特化，静态派发（无运行时开销）
- 编译器可以内联优化
- 可以表达更复杂的类型约束（关联类型、`where` 子句）

代价：
- 每种具体类型都会生成一份特化代码（增加二进制体积）
- 同一个集合中只能包含同一种类型

### 选择指南

```
需要将不同类型的对象放进同一个集合吗？
├── 是 → 使用协议作为存在类型（Boxed Protocol Type）
└── 否 → 需要高级类型约束（关联类型、Self 约束）吗？
    ├── 是 → 使用泛型
    └── 否 → 性能敏感吗？
        ├── 是 → 使用泛型（静态派发）
        └── 否 → 优先使用协议（代码更清晰）

```

### 泛型擦除技术

有时我们需要在泛型和协议之间搭建桥梁。类型擦除（Type Erasure）是常用的技术：

```swift
protocol Shape {
    associatedtype Color
    var color: Color { get }
    func area() -> Double
}

// 类型擦除包装器
struct AnyShape: Shape {
    private let _area: () -> Double

    init<S: Shape>(_ shape: S) {
        _area = { shape.area() }
    }

    func area() -> Double { _area() }
}

// 现在可以将不同类型放入集合
let shapes: [AnyShape] = [
    AnyShape(Circle(radius: 5)),
    AnyShape(Rectangle(width: 3, height: 4))
]
```

### 性能与可读性的平衡

最终的决策往往不是非黑即白的。大多数情况下，**优先使用协议作为公共 API 的抽象手段**，这让代码更具可读性和灵活性。只有在性能瓶颈已经被测量和确认后，再考虑切换到泛型以获得静态派发的性能优势。

```swift
// 推荐的平衡策略：
// 1. 公开 API 使用协议（存在类型）
func process(data source: DataSource) { ... }

// 2. 内部实现使用泛型以获得性能
func processInternal<T: DataSource>(_ source: T) where T.DataType: Codable { ... }
```

## 小结

本章我们讨论了 Swift 开发中最常见的五个设计决策：

| 决策项 | 推荐方向 | 反例 |
|--------|---------|------|
| 值类型 vs 类 | 默认选择值类型 | 需要唯一身份或共享可变状态 |
| 继承 vs 协议 | 默认选择协议 | 需要共享可变状态或 ObjC 兼容 |
| throws vs Result | 边界处用 Result，内部用 throws | 调用方必须处理所有分支 |
| 同步 vs 异步 | 默认同步，需要 I/O 时异步 | 在同步函数中阻塞等待异步 |
| 协议 vs 泛型 | 公共 API 用协议，内部用泛型 | 需要运行时异构集合 |

这些决策不是一成不变的教条，而是基于具体上下文的权衡。Swift 的设计哲学之一是"给开发者选择权"——重要的是理解每个选择的代价和收益，然后做出明智的判断。
