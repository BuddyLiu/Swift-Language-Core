# 第21章 构建一个完整的示例

本章将综合运用全书所学知识，构建一个完整的命令行应用——一个轻量级的股市价监控系统。这个系统聚合多个数据源，展示实时行情，并记录交易历史。

通过这个案例，我们将看到协议导向编程、泛型、值类型、异步并发和错误处理等核心概念如何在实际项目中协同工作。

## 21.1 用协议定义服务边界

好的架构始于清晰的边界。我们先定义系统中所有服务之间的契约协议。

### 数据源协议

```swift
import Foundation

/// 股票信息
public struct Stock: Codable, Equatable, Identifiable, Sendable {
    public let symbol: String
    public let name: String
    public let currency: String

    public var id: String { symbol }

    public init(symbol: String, name: String, currency: String) {
        self.symbol = symbol
        self.name = name
        self.currency = currency
    }
}

/// 行情报价
public struct Quote: Codable, Equatable, Sendable {
    public let symbol: String
    public let price: Decimal
    public let change: Decimal
    public let changePercent: Double
    public let timestamp: Date

    public var isPositive: Bool { change >= 0 }
}

/// 数据源协议 —— 系统的核心抽象
public protocol StockDataSource: AnyObject, Sendable {
    /// 获取该数据源支持的股票列表
    func fetchAvailableStocks() async throws -> [Stock]

    /// 获取指定股票的实时报价
    func fetchQuote(for symbol: String) async throws -> Quote

    /// 数据源的名称（用于日志和展示）
    var sourceName: String { get }
}
```

### 仓储服务

我们定义一个仓储协议来管理用户关注的股票列表：

```swift
/// 关注列表仓储
public protocol WatchlistRepository: AnyObject, Sendable {
    func add(symbol: String) async throws
    func remove(symbol: String) async throws
    func getAll() async throws -> [String]
    func contains(_ symbol: String) async throws -> Bool
}
```

### 日志服务

```swift
/// 日志级别
public enum LogLevel: Comparable, Sendable {
    case debug, info, warning, error

    public var label: String {
        switch self {
        case .debug:   return "DEBUG"
        case .info:    return "INFO"
        case .warning: return "WARN"
        case .error:   return "ERROR"
        }
    }
}

/// 日志服务协议
public protocol LoggingService: AnyObject, Sendable {
    func log(_ message: String, level: LogLevel)
}

extension LoggingService {
    public func debug(_ message: String) { log(message, level: .debug) }
    public func info(_ message: String)  { log(message, level: .info) }
    public func warning(_ message: String) { log(message, level: .warning) }
    public func error(_ message: String) { log(message, level: .error) }
}
```

通过协议定义边界，各个模块间只依赖于抽象，而不依赖具体实现。数据源可以从网络或本地缓存获取数据，关注列表可以存储在内存或文件中——替换实现时不需要修改依赖方的代码。

## 21.2 用值类型建模领域数据

在 Swift 中，值类型（结构体和枚举）是建模领域数据的首选。我们系统的大部分数据模型都使用结构体。

### 聚合数据模型

```swift
/// 带有额外统计信息的报价
public struct DetailedQuote: Equatable, Sendable {
    public let quote: Quote
    public let dayHigh: Decimal
    public let dayLow: Decimal
    public let volume: Int

    public var spread: Decimal { dayHigh - dayLow }
}

/// 交易记录
public struct TradeRecord: Codable, Identifiable, Sendable {
    public let id: UUID
    public let symbol: String
    public let price: Decimal
    public let quantity: Int
    public let timestamp: Date
    public let type: TradeType

    public init(
        id: UUID = UUID(),
        symbol: String,
        price: Decimal,
        quantity: Int,
        timestamp: Date = Date(),
        type: TradeType
    ) {
        self.id = id
        self.symbol = symbol
        self.price = price
        self.quantity = quantity
        self.timestamp = timestamp
        self.type = type
    }
}

/// 交易类型
public enum TradeType: String, Codable, Sendable, CaseIterable {
    case buy  = "买入"
    case sell = "卖出"
}

/// 投资组合中的持仓
public struct Holding: Equatable, Sendable {
    public let symbol: String
    public var quantity: Int
    public var averageCost: Decimal

    public var currentValue: Decimal?
    public var profitLoss: Decimal?

    public mutating func updatePrice(_ price: Decimal) {
        currentValue = Decimal(quantity) * price
        if let cost = currentValue {
            profitLoss = cost - (Decimal(quantity) * averageCost)
        }
    }
}
```

### 使用枚举处理状态

```swift
/// 数据加载状态 —— 利用枚举建模有限状态
public enum LoadingState<T: Sendable>: Sendable {
    case idle
    case loading
    case loaded(T)
    case failed(Error)

    public var value: T? {
        if case .loaded(let v) = self { return v }
        return nil
    }

    public var isLoading: Bool {
        if case .loading = self { return true }
        return false
    }

    public var error: Error? {
        if case .failed(let e) = self { return e }
        return nil
    }
}

/// 市场状态
public enum MarketStatus: String, Sendable {
    case open    = "交易中"
    case closed  = "已休市"
    case preMarket = "盘前"
    case afterHours = "盘后"
}
```

使用值类型建模的好处在于：数据是透明的、可比较的，且不会出现意外的共享状态修改。每个 `TradeRecord` 都是独立的值，复制一份就是完全独立的副本。

## 21.3 泛型装备的可组合网络层

网络层是整个系统的支柱。我们使用泛型设计一个既安全又灵活的网络请求工具。

### 泛型网络客户端

```swift
import Foundation

/// 网络请求错误
public enum NetworkError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int, message: String)
    case decodingFailed(Error)
    case noData

    public var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "无效的 URL"
        case .invalidResponse:
            return "无效的服务器响应"
        case .httpError(let code, let message):
            return "HTTP 错误 \(code): \(message)"
        case .decodingFailed(let error):
            return "数据解码失败: \(error.localizedDescription)"
        case .noData:
            return "服务器未返回数据"
        }
    }
}

/// 可组合的网络请求
public struct NetworkRequest<Response: Decodable & Sendable>: Sendable {
    public let url: URL
    public let method: HTTPMethod
    public let headers: [String: String]
    public let body: Data?
    public let decoder: JSONDecoder
    public let timeout: TimeInterval

    public init(
        url: URL,
        method: HTTPMethod = .get,
        headers: [String: String] = [:],
        body: Data? = nil,
        decoder: JSONDecoder = JSONDecoder(),
        timeout: TimeInterval = 30
    ) {
        self.url = url
        self.method = method
        self.headers = headers
        self.body = body
        self.decoder = decoder
        self.timeout = timeout
    }
}

public enum HTTPMethod: String, Sendable {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
}

/// 泛型网络客户端
public actor NetworkClient {
    private let session: URLSession
    private let baseDecoder: JSONDecoder

    public init(session: URLSession = .shared, decoder: JSONDecoder = JSONDecoder()) {
        self.session = session
        self.baseDecoder = decoder
        // 配置解码器
        baseDecoder.dateDecodingStrategy = .iso8601
        baseDecoder.keyDecodingStrategy = .convertFromSnakeCase
    }

    /// 执行泛型网络请求
    public func execute<Response>(_ request: NetworkRequest<Response>) async throws -> Response
        where Response: Decodable & Sendable
    {
        var urlRequest = URLRequest(url: request.url)
        urlRequest.httpMethod = request.method.rawValue
        urlRequest.allHTTPHeaderFields = request.headers
        urlRequest.httpBody = request.body
        urlRequest.timeoutInterval = request.timeout

        let (data, response) = try await session.data(for: urlRequest)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            let message = String(data: data, encoding: .utf8) ?? "Unknown"
            throw NetworkError.httpError(statusCode: httpResponse.statusCode, message: message)
        }

        do {
            return try request.decoder.decode(Response.self, from: data)
        } catch {
            throw NetworkError.decodingFailed(error)
        }
    }
}
```

### 使用泛型数据源的网络实现

```swift
/// 基于网络的数据源
public final class RemoteStockDataSource: StockDataSource {
    public let sourceName: String
    private let client: NetworkClient
    private let baseURL: URL

    public init(
        sourceName: String = "Remote",
        client: NetworkClient = NetworkClient(),
        baseURL: URL = URL(string: "https://api.example.com")!
    ) {
        self.sourceName = sourceName
        self.client = client
        self.baseURL = baseURL
    }

    public func fetchAvailableStocks() async throws -> [Stock] {
        let url = baseURL.appendingPathComponent("/stocks")
        let request = NetworkRequest<[Stock]>(url: url)
        return try await client.execute(request)
    }

    public func fetchQuote(for symbol: String) async throws -> Quote {
        let url = baseURL.appendingPathComponent("/stocks/\(symbol)/quote")
        let request = NetworkRequest<Quote>(url: url)
        return try await client.execute(request)
    }
}
```

通过泛型参数 `Response`，编译器可以保证每个网络请求的返回类型与解码逻辑完全匹配，且返回值直接是具体类型，不需要调用方再做类型转换。

## 21.4 异步并发下的状态管理

系统需要在多个数据源之间并发获取数据，并优雅地管理状态更新。

### 行情聚合服务

```swift
/// 行情聚合器 —— 从所有数据源获取报价并聚合
public actor QuoteAggregator {
    private let sources: [StockDataSource]

    public init(sources: [StockDataSource]) {
        self.sources = sources
    }

    /// 从所有数据源并行获取报价，返回最快的结果
    public func fetchQuote(for symbol: String) async throws -> Quote {
        try await withThrowingTaskGroup(of: Quote.self) { group in
            for source in sources {
                group.addTask {
                    try await source.fetchQuote(for: symbol)
                }
            }

            // 返回第一个成功的结果
            guard let first = try await group.first(where: { _ in true }) else {
                throw QuoteAggregatorError.noSourceAvailable
            }

            // 取消其余任务
            group.cancelAll()
            return first
        }
    }

    /// 获取关注列表中所有股票的报价（并行）
    public func fetchAllQuotes(for symbols: [String]) async throws -> [String: Quote] {
        try await withThrowingTaskGroup(of: (String, Quote).self) { group in
            for symbol in symbols {
                group.addTask {
                    let quote = try await self.fetchQuote(for: symbol)
                    return (symbol, quote)
                }
            }

            var results: [String: Quote] = [:]
            for try await (symbol, quote) in group {
                results[symbol] = quote
            }
            return results
        }
    }
}

public enum QuoteAggregatorError: Error, LocalizedError {
    case noSourceAvailable

    public var errorDescription: String? {
        switch self {
        case .noSourceAvailable:
            return "所有数据源都不可用"
        }
    }
}
```

### 可观察的状态管理

使用 `@Observable`（Swift 5.9+）或 `@MainActor` 属性包装器来管理 UI 状态：

```swift
import Observation

@MainActor
@Observable
final class PortfolioViewModel {
    private let aggregator: QuoteAggregator
    private let repository: WatchlistRepository
    private let logger: LoggingService

    // 可观察状态
    var holdings: [Holding] = []
    var loadingState: LoadingState<[Holding]> = .idle
    var marketStatus: MarketStatus = .closed

    init(
        aggregator: QuoteAggregator,
        repository: WatchlistRepository,
        logger: LoggingService
    ) {
        self.aggregator = aggregator
        self.repository = repository
        self.logger = logger
    }

    func refreshAll() async {
        loadingState = .loading
        logger.info("开始刷新所有持仓数据")

        do {
            let symbols = try await repository.getAll()
            let quotes = try await aggregator.fetchAllQuotes(for: symbols)

            for i in holdings.indices {
                if let quote = quotes[holdings[i].symbol] {
                    holdings[i].updatePrice(quote.price)
                }
            }

            loadingState = .loaded(holdings)
            logger.info("刷新完成，共 \(holdings.count) 个持仓")
        } catch {
            loadingState = .failed(error)
            logger.error("刷新失败: \(error.localizedDescription)")
        }
    }
}
```

`actor` 保护了聚合器的内部状态不被并发访问破坏，`@Observable` 确保了 UI 能在状态变化时自动刷新。

## 21.5 错误处理与业务流融合

现实系统中的错误往往不是单一的，而是分层的、相关的。我们需要将网络错误、业务错误和系统错误统一处理。

### 定义业务错误

```swift
/// 业务错误
public enum BusinessError: Error, LocalizedError {
    case symbolNotFound(String)
    case marketClosed
    case insufficientQuantity(symbol: String, available: Int, requested: Int)
    case insufficientFunds(available: Decimal, required: Decimal)
    case duplicateSymbol(String)

    public var errorDescription: String? {
        switch self {
        case .symbolNotFound(let symbol):
            return "未找到股票: \(symbol)"
        case .marketClosed:
            return "市场已关闭，无法执行交易"
        case .insufficientQuantity(let symbol, let available, let requested):
            return "\(symbol) 持仓不足：可用 \(available)，请求 \(requested)"
        case .insufficientFunds(let available, let required):
            return "余额不足：可用 \(available)，需要 \(required)"
        case .duplicateSymbol(let symbol):
            return "\(symbol) 已在关注列表中"
        }
    }
}
```

### 业务流中的错误处理

```swift
/// 交易引擎
public actor TradeEngine {
    private let logger: LoggingService

    public init(logger: LoggingService) {
        self.logger = logger
    }

    /// 执行买入操作
    public func buy(
        symbol: String,
        quantity: Int,
        currentPrice: Decimal,
        balance: inout Decimal,
        holdings: inout [Holding]
    ) async -> Result<TradeRecord, BusinessError> {
        let totalCost = currentPrice * Decimal(quantity)

        guard totalCost <= balance else {
            return .failure(.insufficientFunds(available: balance, required: totalCost))
        }

        let record = TradeRecord(
            symbol: symbol,
            price: currentPrice,
            quantity: quantity,
            type: .buy
        )

        // 更新余额
        balance -= totalCost

        // 更新持仓
        if let index = holdings.firstIndex(where: { $0.symbol == symbol }) {
            let existing = holdings[index]
            let totalQuantity = existing.quantity + quantity
            let totalCostOld = Decimal(existing.quantity) * existing.averageCost
            holdings[index] = Holding(
                symbol: symbol,
                quantity: totalQuantity,
                averageCost: (totalCostOld + totalCost) / Decimal(totalQuantity)
            )
        } else {
            holdings.append(Holding(
                symbol: symbol,
                quantity: quantity,
                averageCost: currentPrice
            ))
        }

        logger.info("买入 \(symbol) \(quantity) 股，价格 \(currentPrice)")
        return .success(record)
    }

    /// 执行卖出操作
    public func sell(
        symbol: String,
        quantity: Int,
        currentPrice: Decimal,
        holdings: inout [Holding],
        balance: inout Decimal
    ) async -> Result<TradeRecord, BusinessError> {
        guard let index = holdings.firstIndex(where: { $0.symbol == symbol }) else {
            return .failure(.symbolNotFound(symbol))
        }

        guard holdings[index].quantity >= quantity else {
            return .failure(.insufficientQuantity(
                symbol: symbol,
                available: holdings[index].quantity,
                requested: quantity
            ))
        }

        let record = TradeRecord(
            symbol: symbol,
            price: currentPrice,
            quantity: quantity,
            type: .sell
        )

        // 更新余额和持仓
        balance += currentPrice * Decimal(quantity)
        holdings[index].quantity -= quantity

        // 如果全部卖出，移除持仓
        if holdings[index].quantity == 0 {
            holdings.remove(at: index)
        }

        logger.info("卖出 \(symbol) \(quantity) 股，价格 \(currentPrice)")
        return .success(record)
    }
}
```

在这个设计中，错误与业务流深度融合：每个业务操作都明确地返回 `Result` 类型，调用方必须处理成功和失败两种情形，无法忽略错误。

## 21.6 可测试代码：注入与模拟

最后，但我们保证测试是架构设计中不可或缺的一环。通过依赖注入和协议抽象，我们可以轻松地编写单元测试。

### 模拟数据源

```swift
import XCTest

/// 模拟数据源 —— 用于测试
final class MockStockDataSource: StockDataSource {
    var sourceName: String = "Mock"
    var mockStocks: [Stock] = []
    var mockQuotes: [String: Quote] = [:]
    var shouldThrow = false
    var errorToThrow: Error = NetworkError.noData

    func fetchAvailableStocks() async throws -> [Stock] {
        if shouldThrow { throw errorToThrow }
        return mockStocks
    }

    func fetchQuote(for symbol: String) async throws -> Quote {
        if shouldThrow { throw errorToThrow }
        guard let quote = mockQuotes[symbol] else {
            throw BusinessError.symbolNotFound(symbol)
        }
        return quote
    }
}

/// 模拟日志服务
final class MockLogger: LoggingService {
    var messages: [(String, LogLevel)] = []

    func log(_ message: String, level: LogLevel) {
        messages.append((message, level))
    }
}
```

### 编写单元测试

```swift
final class QuoteAggregatorTests: XCTestCase {
    var sut: QuoteAggregator!
    var mockSource: MockStockDataSource!

    override func setUp() async throws {
        mockSource = MockStockDataSource()
        mockSource.mockQuotes = [
            "AAPL": Quote(symbol: "AAPL", price: 150.0, change: 2.5, changePercent: 1.69, timestamp: Date())
        ]
        sut = QuoteAggregator(sources: [mockSource])
    }

    func test_fetchQuote_success() async throws {
        let quote = try await sut.fetchQuote(for: "AAPL")
        XCTAssertEqual(quote.symbol, "AAPL")
        XCTAssertEqual(quote.price, 150.0)
    }

    func test_fetchQuote_propagatesError() async {
        mockSource.shouldThrow = true

        do {
            _ = try await sut.fetchQuote(for: "AAPL")
            XCTFail("应该抛出错误")
        } catch {
            XCTAssertTrue(error is NetworkError)
        }
    }

    func test_fetchQuote_notFound() async {
        do {
            _ = try await sut.fetchQuote(for: "UNKNOWN")
            XCTFail("应该抛出错误")
        } catch {
            XCTAssertEqual(error as? BusinessError, .symbolNotFound("UNKNOWN"))
        }
    }
}

final class TradeEngineTests: XCTestCase {
    var engine: TradeEngine!
    var logger: MockLogger!

    override func setUp() async throws {
        logger = MockLogger()
        engine = TradeEngine(logger: logger)
    }

    func test_buy_success() async {
        var balance: Decimal = 10000
        var holdings: [Holding] = []

        let result = await engine.buy(
            symbol: "AAPL",
            quantity: 10,
            currentPrice: 150.0,
            balance: &balance,
            holdings: &holdings
        )

        switch result {
        case .success(let record):
            XCTAssertEqual(record.symbol, "AAPL")
            XCTAssertEqual(record.type, .buy)
            XCTAssertEqual(record.quantity, 10)
            XCTAssertEqual(balance, 10000 - 150 * 10)
            XCTAssertEqual(holdings.count, 1)
            XCTAssertEqual(holdings.first?.quantity, 10)
        case .failure:
            XCTFail("买入应该成功")
        }
    }

    func test_buy_insufficientFunds() async {
        var balance: Decimal = 100
        var holdings: [Holding] = []

        let result = await engine.buy(
            symbol: "AAPL",
            quantity: 10,
            currentPrice: 150.0,
            balance: &balance,
            holdings: &holdings
        )

        switch result {
        case .success:
            XCTFail("应该失败")
        case .failure(let error):
            XCTAssertEqual(error, .insufficientFunds(available: 100, required: 1500))
        }
    }

    func test_sell_insufficientQuantity() async {
        var balance: Decimal = 10000
        var holdings: [Holding] = [
            Holding(symbol: "AAPL", quantity: 5, averageCost: 140.0)
        ]

        let result = await engine.sell(
            symbol: "AAPL",
            quantity: 10,
            currentPrice: 150.0,
            holdings: &holdings,
            balance: &balance
        )

        switch result {
        case .success:
            XCTFail("应该失败")
        case .failure(let error):
            XCTAssertEqual(error, .insufficientQuantity(symbol: "AAPL", available: 5, requested: 10))
        }
    }
}
```

### 测试替身的类型

在 Swift 测试中，我们常使用几种测试替身：

- **Stub**：返回固定的预定义数据
- **Mock**：记录调用信息，验证交互行为
- **Fake**：轻量级的真实实现（例如基于字典的内存仓储）
- **Spy**：包装真实对象，记录调用信息

通过协议抽象，我们可以为任意接口提供任意一种测试替身，且编译器保证类型安全。

## 小结

在本章中，我们构建了一个完整的股市监控应用示例，体验了 Swift 核心特性在实际工程中的综合运用：

- **协议**定义了系统的服务边界，实现了模块间的松耦合
- **值类型**安全地建模了领域数据，避免了共享状态带来的问题
- **泛型**构建了可组合的网络层，让类型安全贯穿整个数据流
- **Actor 和结构化并发**管理了复杂的并发状态，避免了数据竞争
- **错误类型和 Result** 将错误处理融入业务流，而不是事后补充
- **依赖注入和模拟**让代码天然可测试，测试成为架构质量的一面镜子

这些实践并非互相独立，而是环环相扣：协议使依赖注入成为可能，值类型简化了状态管理，泛型提升了网络层的复用性，actor 保护了并发安全——每一个设计决策都服务于整体的简洁性和可靠性。
