# 第9章 属性与下标

属性是 Swift 类型中存储和提供数据的方式，而下标则为自定义类型提供了类似数组或字典的便捷访问接口。本章将深入探索属性体系的各个方面，从存储属性到计算属性，从属性观察者到属性包装器，最后到下标——这些特性共同构成了 Swift 数据访问的完整图景。

## 9.1 存储属性与计算属性

Swift 的属性分为两大类：**存储属性**（Stored Properties）和**计算属性**（Computed Properties）。

**存储属性**是存储在实例中的常量或变量，是数据的最基本载体：

```swift
struct Point {
    let x: Double  // 常量存储属性
    var y: Double  // 变量存储属性
}

var point = Point(x: 10, y: 20)
// point.x = 30  // 编译错误：x 是常量
point.y = 30       // 可以修改
```

**计算属性**不直接存储值，而是提供 getter 和可选的 setter 来间接获取和设置其他属性的值：

```swift
struct Rectangle {
    var width: Double
    var height: Double
    
    // 只读计算属性：只有 getter
    var area: Double {
        width * height
    }
    
    // 读写计算属性：getter + setter
    var perimeter: Double {
        get {
            2 * (width + height)
        }
        set {
            // newValue 是默认参数名
            let side = newValue / 4
            width = side
            height = side
        }
    }
    
    // 计算属性也可以自定义参数名
    var diagonal: Double {
        get {
            sqrt(width * width + height * height)
        }
    }
}

var rect = Rectangle(width: 3, height: 4)
print(rect.area)        // 12.0
print(rect.perimeter)   // 14.0

rect.perimeter = 20     // 通过 setter 设置
print(rect.width)       // 5.0
print(rect.height)      // 5.0
```

计算属性的核心特点是**每次访问都重新计算**。这不同于存储属性——存储属性的值存储在内存中，访问时直接读取。理解这一区别对于性能优化至关重要：

```swift
struct Circle {
    var radius: Double
    
    // 每次访问都执行乘法运算
    var area: Double {
        Double.pi * radius * radius
    }
    
    // 不适合频繁调用的场景
    // 如果需要频繁读取 area，应该缓存结果
}
```

计算属性的 setter 可以定义显式的参数名，而非使用默认的 `newValue`：

```swift
struct Temperature {
    var celsius: Double
    
    var fahrenheit: Double {
        get {
            celsius * 9 / 5 + 32
        }
        set(fahrenheitValue) {
            celsius = (fahrenheitValue - 32) * 5 / 9
        }
    }
}

var temp = Temperature(celsius: 100)
print(temp.fahrenheit)           // 212.0
temp.fahrenheit = 98.6
print(temp.celsius)              // 37.0
```

**存储属性和计算属性的本质区别**：

| 特性 | 存储属性 | 计算属性 |
|------|---------|---------|
| 占用内存 | 是 | 否 |
| 可被 `let` 声明 | 是 | 否（必须 `var`） |
| 有观察者 | 是（`willSet`/`didSet`） | 否 |
| 有 setter | 是（可变） | 可选 |
| 执行时机 | 赋值时写入 | 每次访问时计算 |

## 9.2 属性观察者：willSet 与 didSet

属性观察者可以监控存储属性值的变化，在值被设置前后执行代码。这是 Swift 中实现响应式编程的基石之一。

```swift
struct StepCounter {
    var totalSteps: Int = 0 {
        willSet {
            // newValue：即将设置的新值
            print("即将把 totalSteps 设为 \(newValue)")
        }
        didSet {
            // oldValue：原来的旧值
            print("totalSteps 从 \(oldValue) 变为 \(totalSteps)")
            if totalSteps > oldValue {
                print("增加了 \(totalSteps - oldValue) 步")
            }
        }
    }
}

var counter = StepCounter()
counter.totalSteps = 100
// 输出：
// 即将把 totalSteps 设为 100
// totalSteps 从 0 变为 100
// 增加了 100 步

counter.totalSteps = 150
// 输出：
// 即将把 totalSteps 设为 150
// totalSteps 从 100 变为 150
// 增加了 50 步
```

**关键细节**：
1. `willSet` 在值被存储之前调用
2. `didSet` 在值被存储之后调用
3. 即使新旧值相同，观察者也会被触发
4. 在构造器中设置属性不会触发观察者
5. 可以在 `didSet` 中再次修改属性值，不会导致递归调用

```swift
struct User {
    var name: String {
        didSet {
            // 自动确保首字母大写
            if !name.isEmpty && name.first?.isUppercase != true {
                name = name.capitalized
            }
        }
    }
    
    var age: Int {
        didSet {
            // 限制年龄在合理范围
            if age < 0 {
                age = 0
            } else if age > 150 {
                age = 150
            }
        }
    }
}

var user = User(name: "alice", age: 25)
user.name = "bob"         // 自动变为 "Bob"
print(user.name)          // "Bob"
user.age = -5
print(user.age)           // 0
```

**实际应用场景**：

```swift
// 1. 触发 UI 更新
class ViewModel {
    var userName: String = "" {
        didSet {
            updateUI()
        }
    }
    
    private func updateUI() {
        print("UI 更新：\(userName)")
    }
}

// 2. 数据验证与转换
class PasswordField {
    var text: String = "" {
        didSet {
            if text.count < 6 {
                text = oldValue  // 回滚到旧值
                print("密码长度不能少于6位")
            }
        }
    }
}

// 3. 级联更新
struct Temperature {
    var celsius: Double {
        didSet {
            // 保持华氏度与摄氏度同步
            fahrenheit = celsius * 9 / 5 + 32
        }
    }
    var fahrenheit: Double {
        didSet {
            celsius = (fahrenheit - 32) * 5 / 9
        }
    }
    
    init(celsius: Double) {
        self.celsius = celsius
        self.fahrenheit = celsius * 9 / 5 + 32
    }
}
```

## 9.3 惰性属性、全局属性与类型属性

**惰性属性**（Lazy Stored Properties）只在第一次被访问时才进行初始化。这对于初始化开销大或依赖外部条件的属性非常有用：

```swift
class DataImporter {
    init() {
        print("DataImporter 开始初始化...")
        sleep(1)  // 模拟耗时操作
        print("DataImporter 初始化完成")
    }
    
    func importData() -> [String] {
        return ["数据1", "数据2", "数据3"]
    }
}

class DataManager {
    var data: [String] = []
    
    // lazy 属性：只在首次使用时初始化
    lazy var importer = DataImporter()
    
    func loadData() {
        data = importer.importData()
    }
}

let manager = DataManager()  // 此时不会初始化 importer
print("Manager 已创建")
// 输出：Manager 已创建

manager.loadData()           // 第一次访问 importer 时才初始化
// 输出：DataImporter 开始初始化...
//        DataImporter 初始化完成
```

**惰性属性的重要特性**：
1. 必须声明为 `var`，因为初始值可能在初始化后才被设置
2. 是线程不安全的——多线程同时首次访问可能导致多次初始化
3. 如果从未被访问，则永远不会初始化，节省资源
4. 可以在闭包中使用 `self`，因为访问时实例已完全初始化

```swift
class ComplexComputation {
    lazy var expensiveResult: Int = {
        print("开始复杂计算...")
        var result = 0
        for i in 1...100_000_000 {
            result += i
        }
        print("计算完成")
        return result
    }()
}

// 惰性属性的闭包中可以安全地使用 self
class ViewController {
    lazy var actionButton: UIButton = {
        let button = UIButton(type: .system)
        button.addTarget(self, action: #selector(buttonTapped), for: .touchUpInside)
        return button
    }()
    
    @objc func buttonTapped() {
        print("按钮被点击")
    }
}
```

**全局属性**（Global Properties）在文件级别定义，它们默认就是惰性初始化的，且在 Swift 3 之后是线程安全的：

```swift
// 全局常量
let appName = "SwiftBook"

// 全局变量（惰性初始化，线程安全）
var globalConfig: [String: Any] = {
    // 读取配置文件
    return ["version": 1.0, "debug": true]
}()
```

**类型属性**（Type Properties）使用 `static` 关键字，属于类型本身而非实例。它们也是惰性初始化的：

```swift
struct AppSettings {
    // 存储类型属性
    static let appVersion = "2.0.1"
    static var isFirstLaunch = true
    
    // 计算类型属性
    static var fullVersionString: String {
        "\(appName) v\(appVersion)"
    }
    
    // 私有类型属性用于内部共享
    private static let databaseQueue = DispatchQueue(label: "com.app.database")
    
    static func configure() {
        print("配置应用：\(fullVersionString)")
    }
}

print(AppSettings.appVersion)        // "2.0.1"
print(AppSettings.fullVersionString) // "SwiftBook v2.0.1"
AppSettings.configure()              // "配置应用：SwiftBook v2.0.1"
```

**类型属性在类中的特殊支持**：类还可以使用 `class` 关键字声明可被子类重写的计算类型属性：

```swift
class Vehicle {
    class var maxSpeed: Int {
        return 120
    }
    
    class var description: String {
        return "这是一辆交通工具，最高速度 \(maxSpeed) km/h"
    }
}

class Car: Vehicle {
    override class var maxSpeed: Int {
        return 200
    }
}

print(Vehicle.description)  // "这是一辆交通工具，最高速度 120 km/h"
print(Car.description)      // "这是一辆交通工具，最高速度 200 km/h"
```

## 9.4 属性包装器（Property Wrapper）：@Binding、@State 的原理

属性包装器（Property Wrapper）是 Swift 5.1 引入的强大特性，它允许你将属性的存取逻辑封装到一个可复用的包装类型中。SwiftUI 中的 `@State`、`@Binding`、`@Published` 等核心属性包装器就是基于这一机制实现的。

**基本语法**：

```swift
// 1. 定义属性包装器
@propertyWrapper
struct Clamped<T: Comparable> {
    private var value: T
    private let min: T
    private let max: T
    
    init(wrappedValue: T, min: T, max: T) {
        self.min = min
        self.max = max
        self.value = Swift.min(Swift.max(wrappedValue, min), max)
    }
    
    var wrappedValue: T {
        get { value }
        set { value = Swift.min(Swift.max(newValue, min), max) }
    }
}

// 2. 使用属性包装器
struct GameSettings {
    @Clamped(min: 0, max: 100) var volume: Int = 50
    @Clamped(min: 1, max: 10) var difficulty: Int = 5
}

var settings = GameSettings()
print(settings.volume)         // 50
settings.volume = 200
print(settings.volume)         // 100 — 被限制在最大值
settings.volume = -10
print(settings.volume)         // 0 — 被限制在最小值
```

**属性包装器的投影值**：通过 `projectedValue` 提供额外的接口，SwiftUI 的 `@Binding` 就是通过投影值实现的：

```swift
@propertyWrapper
struct UserDefault<T> {
    private let key: String
    private let defaultValue: T
    
    init(wrappedValue: T, _ key: String) {
        self.key = key
        self.defaultValue = wrappedValue
    }
    
    var wrappedValue: T {
        get {
            UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        set {
            UserDefaults.standard.set(newValue, forKey: key)
        }
    }
    
    // 投影值：提供额外的能力
    var projectedValue: Self { return self }
    
    func reset() {
        UserDefaults.standard.removeObject(forKey: key)
    }
}

struct AppConfig {
    @UserDefault("is_logged_in") var isLoggedIn: Bool = false
    @UserDefault("user_name") var userName: String = "Guest"
}

var config = AppConfig()
config.isLoggedIn = true
config.userName = "Alice"

// 通过 $ 访问投影值
print(config.$isLoggedIn)  // UserDefault 实例
config.$isLoggedIn.reset() // 重置为默认值
```

**理解 SwiftUI 中的 @State 和 @Binding**：

SwiftUI 中的 `@State` 本质上是将属性存储在 SwiftUI 框架管理的存储中，并在值变化时自动刷新视图：

```swift
// @State 的简化原理
@propertyWrapper
struct State<Value> {
    private var storage: Value
    
    init(wrappedValue: Value) {
        self.storage = wrappedValue
    }
    
    var wrappedValue: Value {
        get { storage }
        set {
            storage = newValue
            // 通知 SwiftUI 刷新相关视图
            // ...
        }
    }
    
    // 投影值返回 Binding
    var projectedValue: Binding<Value> {
        Binding(get: { self.wrappedValue },
                set: { self.wrappedValue = $0 })
    }
}

// 使用方式
struct CounterView: View {
    @State private var count = 0
    
    var body: some View {
        VStack {
            Text("计数：\(count)")
            Button("增加") { count += 1 }
            // $count 是投影值，即 Binding<Int>
            ChildView(value: $count)
        }
    }
}

struct ChildView: View {
    @Binding var value: Int
    
    var body: some View {
        Button("子视图修改：\(value)") {
            value += 1
        }
    }
}
```

**自定义属性包装器实战**：

```swift
import Foundation

// 1. 线程安全属性包装器
@propertyWrapper
struct ThreadSafe<T> {
    private let queue = DispatchQueue(label: "com.threadsafe", attributes: .concurrent)
    private var value: T
    
    init(wrappedValue: T) {
        self.value = wrappedValue
    }
    
    var wrappedValue: T {
        get {
            queue.sync { value }
        }
        mutating set {
            queue.sync(flags: .barrier) { value = newValue }
        }
    }
}

// 2. 过期缓存包装器
@propertyWrapper
struct Cached<T> {
    private var value: T?
    private let duration: TimeInterval
    private var timestamp: Date = .distantPast
    
    init(wrappedValue: T?, duration: TimeInterval) {
        self.value = wrappedValue
        self.duration = duration
    }
    
    var wrappedValue: T? {
        mutating get {
            if Date().timeIntervalSince(timestamp) > duration {
                value = nil  // 缓存过期
            }
            return value
        }
        set {
            value = newValue
            timestamp = Date()
        }
    }
}
```

## 9.5 下标：像数组一样访问自定义类型

下标允许自定义类型通过中括号语法（`[]`）访问其元素。这极大地提高了代码的可读性和便利性。

**基本语法**：

```swift
struct Matrix {
    private var grid: [Double]
    let rows: Int
    let columns: Int
    
    init(rows: Int, columns: Int) {
        self.rows = rows
        self.columns = columns
        self.grid = Array(repeating: 0.0, count: rows * columns)
    }
    
    // 下标方法
    subscript(row: Int, column: Int) -> Double {
        get {
            assert(indexIsValid(row: row, column: column), "索引越界")
            return grid[row * columns + column]
        }
        set {
            assert(indexIsValid(row: row, column: column), "索引越界")
            grid[row * columns + column] = newValue
        }
    }
    
    private func indexIsValid(row: Int, column: Int) -> Bool {
        row >= 0 && row < rows && column >= 0 && column < columns
    }
}

var matrix = Matrix(rows: 3, columns: 3)
matrix[0, 0] = 1.0
matrix[1, 1] = 1.0
matrix[2, 2] = 1.0

print(matrix[0, 0])  // 1.0
print(matrix[0, 1])  // 0.0
```

**下标可以有多个参数**，参数类型可以是任意类型：

```swift
struct Grid<T> {
    private var items: [T]
    let rows: Int
    let columns: Int
    
    init(rows: Int, columns: Int, defaultValue: T) {
        self.rows = rows
        self.columns = columns
        self.items = Array(repeating: defaultValue, count: rows * columns)
    }
    
    // 多个 Int 参数
    subscript(row: Int, column: Int) -> T {
        get { items[row * columns + column] }
        set { items[row * columns + column] = newValue }
    }
    
    // 使用 Range 作为参数
    subscript(row: Int, columnRange: Range<Int>) -> [T] {
        columnRange.map { items[row * columns + $0] }
    }
}

var grid = Grid(rows: 4, columns: 4, defaultValue: 0)
grid[0, 0] = 1
grid[0, 1] = 2
grid[0, 2] = 3
print(grid[0, 0..<3])  // [1, 2, 3]
```

**类型下标**：Swift 5.1 开始支持静态下标：

```swift
enum FileExtension: String {
    case swift = ".swift"
    case md = ".md"
    case json = ".json"
    case plist = ".plist"
    
    // 类型下标：通过扩展名字符串获取对应的枚举值
    static subscript(_ ext: String) -> FileExtension? {
        return FileExtension(rawValue: ext)
    }
}

let ext = FileExtension["swift"]
print(ext == .swift)  // true
```

**下标实战**：实现一个安全的字典包装器

```swift
struct SafeDictionary<Key: Hashable, Value> {
    private var dictionary: [Key: Value]
    private let defaultValue: Value
    
    init(defaultValue: Value) {
        self.dictionary = [:]
        self.defaultValue = defaultValue
    }
    
    // 安全下标：键不存在时返回默认值，而不是 nil
    subscript(key: Key) -> Value {
        get {
            dictionary[key] ?? defaultValue
        }
        set {
            dictionary[key] = newValue
        }
    }
    
    // 下标带默认参数版本
    subscript(key: Key, default default: Value) -> Value {
        get {
            dictionary[key] ?? `default`
        }
        set {
            dictionary[key] = newValue
        }
    }
}

var dict = SafeDictionary<String, Int>(defaultValue: 0)
dict["a"] = 100
print(dict["a"])        // 100
print(dict["b"])        // 0 — 使用默认值
print(dict["b", default: -1])  // -1 — 使用自定义默认值
```

**下标的灵活运用**：

```swift
// 1. 字符串下标扩展
extension String {
    subscript(index: Int) -> Character? {
        guard index >= 0 && index < count else { return nil }
        return self[self.index(startIndex, offsetBy: index)]
    }
    
    subscript(range: ClosedRange<Int>) -> String {
        let start = self.index(startIndex, offsetBy: range.lowerBound)
        let end = self.index(startIndex, offsetBy: range.upperBound)
        return String(self[start...end])
    }
}

let str = "Hello, Swift!"
print(str[0]!)     // "H"
print(str[7...11]) // "Swift"

// 2. 二维数组下标
struct ChessBoard {
    private var board: [[String]]
    
    init() {
        board = Array(repeating: Array(repeating: ".", count: 8), count: 8)
        // 初始化棋子位置...
    }
    
    // 支持类似 chessBoard["e2"] 的下标
    subscript(position: String) -> String? {
        guard position.count == 2 else { return nil }
        guard let col = position.first?.asciiValue.map({ Int($0 - 97) }),
              let row = Int(String(position.last!)), (1...8).contains(row) else {
            return nil
        }
        let rowIndex = 8 - row
        guard rowIndex >= 0 && rowIndex < 8 && col >= 0 && col < 8 else { return nil }
        return board[rowIndex][col]
    }
}

var board = ChessBoard()
print(board["e2"] ?? "")  // 获取 e2 位置的棋子
```

属性与下标是 Swift 数据访问的两大支柱。从基础的存储属性到高级的属性包装器，从简单的 getter/setter 到多维下标，这些特性赋予了开发者极大的灵活性和表达力。深入理解它们，你将能设计出既安全又优雅的 API 接口。
