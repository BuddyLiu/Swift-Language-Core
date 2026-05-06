# 第18章 迁移过去：从 GCD 到结构化并发

## 18.1 队列与事件循环

在结构化并发出现之前，Grand Central Dispatch（GCD）是 Swift/Objective-C 生态中并发编程的主流工具。GCD 基于**派发队列（Dispatch Queue）** 和 **事件循环（Run Loop）** 模型，将任务交给系统管理的线程池执行。

### GCD 的核心机制

```swift
// 串行队列：任务一次执行一个
let serialQueue = DispatchQueue(label: "com.example.serial")
serialQueue.async {
    print("任务 1")
}
serialQueue.async {
    print("任务 2")
}

// 并发队列：多个任务同时执行
let concurrentQueue = DispatchQueue(label: "com.example.concurrent",
                                    attributes: .concurrent)
concurrentQueue.async {
    print("并发任务 A")
}
concurrentQueue.async {
    print("并发任务 B")
}
```

GCD 的运作依赖操作系统调度：每个队列关联一个线程池，系统根据 CPU 核心数和负载动态调整线程数。开发者通过队列类型（串行/并发）和 QoS（Quality of Service）控制优先级。

### 事件循环与 RunLoop

在主线程上，RunLoop 负责处理输入事件、定时器和派发 block。每个线程都有其 RunLoop，但仅在需要时创建。异步回调本质上是通过 RunLoop 的 timer source 或 dispatch source 唤醒线程。

### GCD 的局限性

GCD 虽然灵活，但在大型项目中暴露出几个核心问题：

1. **作用域分离**：任务的创建和执行在闭包中隐式进行，容易导致生命周期混乱。
2. **错误处理困难**：错误必须在回调闭包中手动传递。
3. **缺乏取消机制**：一旦派发，无法优雅取消正在等待或执行的任务。
4. **数据竞争**：GCD 不提供任何数据保护机制，需要开发者自己加锁。

## 18.2 传统异步回调的问题

### 回调地狱

多层嵌套的异步回调使代码难以阅读和维护：

```swift
func processUserData(userID: String, completion: @escaping (Result<Profile, Error>) -> Void) {
    fetchUser(userID) { user in
        fetchPosts(user.id) { posts in
            fetchAvatar(posts.first?.imageURL) { image in
                guard let image = image else {
                    completion(.failure(ProfileError.missingAvatar))
                    return
                }
                let profile = Profile(user: user, posts: posts, avatar: image)
                completion(.success(profile))
            }
        }
    }
}
```

这段代码存在三个问题：
- 每一层缩进增加认知负荷。
- 错误处理在每个回调中重复。
- 如果某个中间步骤失败，前面的计算资源无法自动回收。

### 资源管理风险

回调式异步编程中，`completion` 闭包可能被多次调用、漏调或逃逸到错误的线程：

```swift
func loadResource(completion: @escaping (Resource?) -> Void) {
    DispatchQueue.global().async {
        // 执行耗时操作
        let result = expensiveWork()
        DispatchQueue.main.async {
            // 可能忘记调用 completion
            // 也可能在多个路径下重复调用
            if result.isValid {
                completion(result) // 路径一
                return
            }
            completion(nil) // 路径二
            // 如果 isValid 为 false 但忘记 return，两个 completion 都会执行
        }
    }
}
```

## 18.3 渐进式迁移策略

将现有 GCD 代码迁移到结构化并发应该采取渐进策略，而非一次性重写。下面推荐分阶段的迁移方法。

### 第一阶段：封装异步函数

将基于回调的 API 封装为 `async` 函数，使其可与 `await` 配合使用。

```swift
// 改造前：基于回调
func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
    // ... GCD 实现
}

// 改造后：async 封装
func fetchData() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        fetchData { result in
            continuation.resume(with: result)
        }
    }
}
```

`withCheckedThrowingContinuation`（以及非抛出版本 `withCheckedContinuation`）是连接回调世界和 async/await 世界的桥梁。它会在编译期和运行时检查 `continuation` 是否恰好被 resume 一次。

### 第二阶段：替换串行队列为 Actor

串行队列常用于保护共享可变状态。Actor 提供编译期检查的安全替代方案。

```swift
// GCD 方式
class AccountManager {
    private let queue = DispatchQueue(label: "account")
    private var accounts: [String: Account] = [:]
    
    func addAccount(_ account: Account) {
        queue.async { self.accounts[account.id] = account }
    }
    
    func getAccount(id: String, completion: @escaping (Account?) -> Void) {
        queue.async { completion(self.accounts[id]) }
    }
}

// Actor 方式
actor AccountManager {
    private var accounts: [String: Account] = [:]
    
    func addAccount(_ account: Account) {
        accounts[account.id] = account
    }
    
    func getAccount(id: String) -> Account? {
        return accounts[id]
    }
}
```

### 第三阶段：使用 TaskGroup 替换 DispatchGroup

```swift
// GCD 方式
func loadAll(completion: @escaping ([Data]) -> Void) {
    let group = DispatchGroup()
    var results: [Data] = []
    
    for url in urls {
        group.enter()
        load(url) { data in
            results.append(data)
            group.leave()
        }
    }
    
    group.notify(queue: .main) {
        completion(results)
    }
}

// Async 方式
func loadAll() async -> [Data] {
    await withTaskGroup(of: Data.self) { group in
        for url in urls {
            group.addTask { await load(url) }
        }
        
        var results: [Data] = []
        for await data in group {
            results.append(data)
        }
        return results
    }
}
```

### 第四阶段：整体模块迁移

当项目中核心模块完成 API 封装后，可以逐步将调用方从回调模式迁移到 async/await 模式。建议以特性为单位，自底向上迁移——先迁移底层基础设施 API，再迁移业务逻辑层，最后迁移 UI 层。

## 18.4 并发下的值类型优势

在迁移到结构化并发的过程中，值类型（`struct`、`enum`）的优势更加凸显。

### 不可变性保证

```swift
struct Order {
    let id: String
    let items: [Item]
    let totalPrice: Decimal
}

// 多个 Task 可以安全地共享 Order 的引用
func processOrders(_ orders: [Order]) async {
    await withTaskGroup(of: Receipt.self) { group in
        for order in orders {
            group.addTask {
                // 每个 Task 获得 order 的独立副本
                return await generateReceipt(for: order)
            }
        }
    }
}
```

值类型的每次传递都是独立的副本，这天然消除了数据竞争的可能性。相比之下，引用类型（`class`）的多个 Task 共享同一实例，需要额外的同步机制。

### 优先选择值类型

在设计并发系统时，应优先使用值类型表示数据传输对象（DTO）和不可变配置：

```swift
// 推荐：值类型，并发安全
struct UserProfile {
    let name: String
    let bio: String
    let avatarURL: URL
}

// 需要数据共享时才使用引用类型 + Actor
actor SharedSession {
    var cache: [String: Data] = [:]
    var activeRequests: Int = 0
}
```

### Sendable 一致性

值类型在满足条件时自动遵守 `Sendable` 协议，这使它们可以自由跨越并发域传递。在迁移过程中，将 `class` 改为 `struct` 通常是消除编译器警告的最直接方式：

```swift
// 迁移前：class，需关注 Sendable
class Configuration {
    let timeout: TimeInterval
    let retryCount: Int
}

// 迁移后：struct，自动 Sendable
struct Configuration: Sendable {
    let timeout: TimeInterval
    let retryCount: Int
}
```

## 小结

从 GCD 到结构化并发的迁移不仅是语法的更新，更是一次并发思维方式的变革。通过将并发任务与代码作用域绑定、提供编译期安全检查以及简化的错误处理模型，结构化并发使 Swift 代码更安全、更易维护。渐进式迁移策略降低了过渡风险，而值类型和 Sendable 的配合使用则构建了数据安全的坚实基石。下一章我们将探讨 Swift 与传统 Cocoa 生态的桥梁——与 Objective-C 的互操作性。
