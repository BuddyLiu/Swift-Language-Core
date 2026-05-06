# 第17章 结构化并发（Swift 5.5+）

## 17.1 async/await 核心理念

Swift 5.5 引入的 async/await 机制标志着 Swift 并发编程范式的根本性转变。这一特性源于"结构化并发"（Structured Concurrency）思想：并发任务的生命周期应当与代码的语法结构一一对应，而非在回调闭包中隐式传递。

### 从回调到同步写法

传统的异步编程依赖闭包回调或代理模式，导致所谓的"回调地狱"——控制流被打散到多个独立的闭包中，错误处理和资源管理变得脆弱。

```swift
// 传统回调方式
func fetchUserData(completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, error in
        if let error = error {
            completion(.failure(error))
            return
        }
        completion(.success(parseUser(data)))
    }.resume()
}

// async/await 方式
func fetchUserData() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return parseUser(data)
}
```

`async` 关键字标记函数为异步函数，`await` 则用于挂起当前任务直至异步操作完成。挂起期间当前线程不会被阻塞，可以执行其他任务——这类似于协程的机制。

### 异步函数的语义

- **async**：函数声明中的修饰符，表示该函数可能在其执行过程中挂起。
- **await**：调用 `async` 函数时的挂起点，当前任务在此暂停，系统调度恢复执行。

```swift
func loadImage() async throws -> UIImage {
    let url = URL(string: "https://example.com/image.png")!
    let (data, _) = try await URLSession.shared.data(from: url)
    guard let image = UIImage(data: data) else {
        throw LoadingError.invalidData
    }
    return image
}
```

`await` 关键字还起到了文档作用——代码读者可直观看出哪些地方可能发生挂起，从而理解潜在的并发行为。

## 17.2 Task 与 TaskGroup

### Task：基本并发单元

`Task` 是结构化并发的核心构建块，表示一个可独立执行的异步工作单元。

```swift
// 创建并启动一个 Task
Task {
    let image = try await loadImage()
    await MainActor.run {
        imageView.image = image
    }
}
```

`Task` 继承创建它的上下文（优先级、Actor 隔离等），并在继承的 Actor 或全局并发池上执行。

### 获取异步结果

`Task` 可以返回结果，通过 `await` 获取：

```swift
let handle = Task {
    return try await fetchUserData()
}

let user = try await handle.value
```

### TaskGroup：结构化并发组合

当需要并发执行多个独立任务并等待所有任务完成时，`TaskGroup` 提供结构化的管理方式。

```swift
func loadAllImages(_ urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        for url in urls {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                guard let image = UIImage(data: data) else {
                    throw LoadingError.invalidData
                }
                return image
            }
        }
        
        var images: [UIImage] = []
        for try await image in group {
            images.append(image)
        }
        return images
    }
}
```

`withThrowingTaskGroup` 确保所有子任务在作用域结束时被正确清理——不论正常完成还是抛出错误。这就是"结构化"的含义：任务的生命周期严格绑定于代码作用域。

### 未捕获的错误传播

在 TaskGroup 中，任何子任务抛出的错误都会通过 `for try await` 循环传播。第一个抛出的错误会隐式取消组中所有剩余子任务。

## 17.3 Actor：隔离共享可变状态

并发编程中最棘手的问题之一是对共享可变状态的竞争访问。传统的解决方案是锁或串行队列，但都容易出错。Swift 的 **Actor** 提供编译器强制保护的数据隔离机制。

### Actor 的基本语法

```swift
actor BankAccount {
    private var balance: Double
    
    init(initialBalance: Double) {
        self.balance = initialBalance
    }
    
    func deposit(_ amount: Double) {
        balance += amount
    }
    
    func withdraw(_ amount: Double) -> Bool {
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }
    
    var currentBalance: Double {
        balance
    }
}
```

Actor 的所有属性和方法默认只能从 Actor 内部访问。外部访问必须通过 `await`：

```swift
let account = BankAccount(initialBalance: 1000)
await account.deposit(500)
let balance = await account.currentBalance
print(balance) // 1500
```

### Actor 重入

Actor 在执行 `await` 挂起点时允许其他任务进入 Actor——这称为 **Actor 重入（Actor Reentrancy）**。这意味着 Actor 的内部状态在挂起前后可能发生变化。

```swift
actor DataProcessor {
    var state: State = .idle
    
    func process() async {
        state = .processing
        // 挂起点：其他任务可以在此处修改 state
        let result = await expensiveComputation()
        // 此时 state 可能已被改变！
        state = .completed(result)
    }
}
```

重入是必要的设计，防止 Actor 死锁。但开发者需要意识到这一点，在挂起操作前后仔细验证 Actor 的不变性。

## 17.4 Sendable 协议

并发环境中传递值必须保证数据安全。**Sendable** 协议标记一个类型可以安全地在并发域间传递。

### 编译器检查

```swift
struct Person: Sendable {
    let name: String
    let age: Int
}

class MutablePerson { // 编译错误：class 没有显式标记为 @unchecked Sendable
    var name: String
    init(name: String) { self.name = name }
}
```

值类型（struct、enum）如果所有存储属性都符合 Sendable，则自动隐式遵守 Sendable。对于 class，则需要显式声明，且必须是 final 且所有属性不可变。

### @unchecked Sendable

对于无法满足编译器静态检查但开发者确知安全的类型，可以使用 `@unchecked Sendable`：

```swift
@unchecked Sendable
class ThreadSafeCache: @unchecked Sendable {
    private let queue = DispatchQueue(label: "cache")
    private var storage: [String: Data] = [:]
    
    func get(_ key: String) -> Data? {
        queue.sync { storage[key] }
    }
    
    func set(_ key: String, _ value: Data) {
        queue.async { self.storage[key] = value }
    }
}
```

`@unchecked Sendable` 是将责任转移给开发者，应当谨慎使用。

## 17.5 任务取消与协作

Swift 结构化并发中的任务取消采用**协作式（Cooperative）** 模式。外部可以请求取消一个任务，但任务本身需要定期检查取消状态并做出响应。

### 取消检查

```swift
func downloadLargeFile() async throws {
    let url = URL(string: "https://example.com/largefile")!
    let (stream, _) = try await URLSession.shared.bytes(from: url)
    
    var buffer = Data()
    for try await chunk in stream {
        // 检查取消
        try Task.checkCancellation()
        
        buffer.append(chunk)
        // 或者使用 Task.isCancelled 进行非抛出的检查
        guard !Task.isCancelled else {
            // 优雅清理
            return
        }
    }
}
```

`Task.checkCancellation()` 会在任务被取消时抛出 `CancellationError`，配合 `try` 可立即终止当前函数。`Task.isCancelled` 属性则用于需要自定义清理逻辑的场景。

### 取消传播

取消是结构化的：取消一个父任务会自动传播到所有子任务。

```swift
let parent = Task {
    let child1 = Task { await work() }
    let child2 = Task { await work() }
    // 如果 parent 被取消，child1 和 child2 也将被取消
}
```

### 处理取消通知

```swift
Task {
    await withTaskCancellationHandler {
        try await performWork()
    } onCancel: {
        // 取消时的清理操作，不在此切换线程
        cleanup()
    }
}
```

## 17.6 @MainActor 主线程交互

UI 更新必须在主线程上执行。Swift 的 `@MainActor` 属性包装器将函数或类型的执行隔离到主 Actor（即主线程），编译器确保这一点得到遵守。

### 使用 @MainActor

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    
    func updateItems() async {
        let newItems = await fetchItems()
        items = newItems // 自动在主 Actor 上执行
    }
}
```

标记为 `@MainActor` 的类型，其所有属性和方法都在主线程上执行。外部调用需要使用 `await`：

```swift
let vm = ViewModel()
await vm.updateItems()
```

### 全局 Actor

`@MainActor` 是 Swift 标准库提供的全局 Actor。它的本质是 `@globalActor` 加上一个共享的 Actor 实例。开发者也可以定义自己的全局 Actor：

```swift
@globalActor actor MyBackgroundActor {
    static let shared = MyBackgroundActor()
}

@MyBackgroundActor
func backgroundWork() {
    // 在自定义 Actor 上执行
}
```

### 与 UIKit/SwiftUI 集成

在 SwiftUI 中，`@State`、`@ObservedObject` 等属性包装器已经隐式与 `@MainActor` 关联。在 View 的 `body` 属性中，你通常无需显式标记 `@MainActor`——SwiftUI 框架已经处理了主线程调度。

```swift
struct ContentView: View {
    @State private var data: [String] = []
    
    var body: some View {
        List(data, id: \.self) { item in
            Text(item)
        }
        .task {
            // task 修饰符自动在主 Actor 上执行
            data = await loadData()
        }
    }
}
```

## 小结

结构化并发为 Swift 带来了可预测、可组合的并发模型。`async/await` 简化了异步代码的书写与理解，`Task` 和 `TaskGroup` 提供了结构化任务管理，`Actor` 从根本上消除了数据竞争，`Sendable` 确保数据传递安全，而 `@MainActor` 使主线程交互变得简洁而安全。这一整套工具共同构筑了 Swift 在并发领域的现代实践基础。下一章我们将探讨如何从传统的 GCD 模式迁移到结构化并发。
