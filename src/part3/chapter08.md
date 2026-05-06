# 第8章 类的正确使用场景

在上一章中，我们深入探讨了结构体和枚举这些值类型的优势。然而，有一类问题场景是值类型无法很好解决的，例如需要共享状态、需要同一性判断、需要与 Objective-C 互操作等。在这些场景下，类（class）依然是不可替代的选择。本章将深入剖析引用语义的特性，帮助你理解何时以及如何正确地使用类。

## 8.1 引用语义与身份：=== 身份运算符

类的本质是**引用类型**。当你将一个类的实例赋值给多个变量时，这些变量都指向内存中的同一个对象。

```swift
class User {
    var name: String
    var points: Int
    
    init(name: String, points: Int) {
        self.name = name
        self.points = points
    }
}

let user1 = User(name: "Alice", points: 100)
let user2 = user1  // user2 和 user1 指向同一个对象

user2.name = "Bob"
print(user1.name)  // "Bob" — 因为 user1 和 user2 是同一个实例
```

对于引用类型，Swift 提供了**身份运算符**（Identity Operators）`===` 和 `!==`，用于判断两个引用是否指向同一个对象。这与 `==`（判断值相等）有本质区别：

```swift
let userA = User(name: "Alice", points: 100)
let userB = User(name: "Alice", points: 100)
let userC = userA

print(userA === userB)  // false — 不同实例，即便内容相同
print(userA === userC)  // true — 同一实例
print(userA !== userB)  // true

// 要让 == 可用，需要让类遵循 Equatable 协议
extension User: Equatable {
    static func == (lhs: User, rhs: User) -> Bool {
        lhs.name == rhs.name && lhs.points == rhs.points
    }
}

print(userA == userB)   // true — 内容相等
print(userA == userC)   // true — 内容当然也相等
```

**什么时候需要 `===`？** 典型场景是在集合中查找特定对象，或者实现观察者模式时移除特定的观察者：

```swift
class Observer {
    let id: UUID
    init() { self.id = UUID() }
}

class Observable {
    private var observers: [Observer] = []
    
    func addObserver(_ observer: Observer) {
        observers.append(observer)
    }
    
    func removeObserver(_ observer: Observer) {
        // 使用 === 而不是 ==，确保移除的是精确的同一个对象
        observers.removeAll { $0 === observer }
    }
}
```

需要注意的是，结构体不支持 `===` 运算符，因为结构体实例本身没有持久化的"身份"概念——每次拷贝都是全新的值。这正是引用类型和值类型的哲学分野所在。

## 8.2 继承的局限与替代：继承 vs 协议组合

类是 Swift 中唯一支持继承的类型。继承确实能带来代码复用，但它也有着众所周知的局限：

**继承的局限性：**
- 单一继承限制：一个类只能有一个父类
- 脆弱的基类问题：修改父类可能影响所有子类
- 菱形继承问题：虽然 Swift 通过单一继承避免了此问题，但限制了建模能力
- 继承层次过深导致代码难以理解和维护

```swift
// 继承的典型问题：层次结构僵化
class Animal {
    func makeSound() { }
}

class Dog: Animal {
    override func makeSound() { print("汪汪") }
}

class Cat: Animal {
    override func makeSound() { print("喵喵") }
}

// 如果我们需要一个既会狗叫又会猫叫的动物怎么办？
// 或者需要一个不会叫的动物？
// 继承体系难以灵活应对这些需求变化
```

**协议组合**提供了更灵活的替代方案。Swift 的协议可以多继承（一个类型可以遵循多个协议），结合协议扩展可以提供默认实现：

```swift
protocol Runnable {
    func run()
}

protocol Swimmable {
    func swim()
}

protocol Flyable {
    func fly()
}

// 通过协议扩展提供默认实现
extension Runnable {
    func run() { print("跑步前进") }
}

extension Swimmable {
    func swim() { print("游泳前进") }
}

// 一个类型可以同时遵循多个协议
struct Duck: Runnable, Swimmable, Flyable {
    // 只需实现 Flyable，其他的使用默认实现
    func fly() { print("飞翔") }
}

struct Dog: Runnable, Swimmable {
    func run() { print("狗在奔跑") }
}

struct Fish: Swimmable { }
```

使用协议组合时，**面向协议编程**（POP）推荐的做法是小而专注的协议。这与面向对象编程中庞大的基类形成鲜明对比：

```swift
// 面向对象方式：大型基类
class DataSourceBase {
    func fetch() -> [String] { return [] }
    func save(_ items: [String]) { }
    func validate(_ item: String) -> Bool { return true }
    func transform(_ item: String) -> String { return item }
}

// 面向协议方式：分离关注点
protocol Fetchable {
    associatedtype Item
    func fetch() -> [Item]
}

protocol Saveable {
    associatedtype Item
    func save(_ items: [Item])
}

protocol Validatable {
    associatedtype Item
    func validate(_ item: Item) -> Bool
}

protocol Transformable {
    associatedtype Item
    func transform(_ item: Item) -> Item
}

// 按需组合
struct UserRepository: Fetchable, Saveable {
    typealias Item = String
    
    func fetch() -> [String] {
        return ["Alice", "Bob"]
    }
    
    func save(_ items: [String]) {
        print("保存用户：\(items)")
    }
}
```

**何时仍然需要继承？**
- 需要利用 Objective-C 的动态派发和 KVO 时
- 使用 UIKit/AppKit 等框架时（这些框架大量使用继承）
- 需要实现类簇（Class Cluster）模式时
- 多个子类共享状态（而非行为）时

## 8.3 类型转换：is 与 as

Swift 是强类型语言，但有时我们需要在类型层次中进行检查和转换。Swift 提供了 `is`、`as?`、`as!` 和 `as` 运算符来处理类型转换。

**`is` 运算符**：检查实例是否属于某种类型：

```swift
class MediaItem {
    var title: String
    init(title: String) { self.title = title }
}

class Movie: MediaItem {
    var director: String
    init(title: String, director: String) {
        self.director = director
        super.init(title: title)
    }
}

class Song: MediaItem {
    var artist: String
    init(title: String, artist: String) {
        self.artist = artist
        super.init(title: title)
    }
}

let library: [MediaItem] = [
    Movie(title: "星际穿越", director: "诺兰"),
    Song(title: "晴天", artist: "周杰伦"),
    Movie(title: "盗梦空间", director: "诺兰"),
    Song(title: "七里香", artist: "周杰伦")
]

var movieCount = 0
var songCount = 0

for item in library {
    if item is Movie {
        movieCount += 1
    } else if item is Song {
        songCount += 1
    }
}
print("电影：\(movieCount)，歌曲：\(songCount)")  // 电影：2，歌曲：2
```

**向下转型**：`as?` 和 `as!`

```swift
for item in library {
    // 安全转型 as? — 转型失败返回 nil
    if let movie = item as? Movie {
        print("电影：\(movie.title)，导演：\(movie.director)")
    } else if let song = item as? Song {
        print("歌曲：\(song.title)，歌手：\(song.artist)")
    }
}

// 强制转型 as! — 仅在你确定类型正确时使用
let firstMovie = library[0] as! Movie
print(firstMovie.director)  // "诺兰"
```

**`as` 向上转型**：将子类实例视为父类类型，这是安全的，所以不需要可选形式：

```swift
let movie = Movie(title: "信条", director: "诺兰")
let mediaItem = movie as MediaItem  // 向上转型，总是安全的
```

**Swift 中类型转换的典型应用场景**：

```swift
// 1. 处理异构数组
let mixed: [Any] = ["Hello", 42, 3.14, Movie(title: "ABC", director: "DEF")]

for element in mixed {
    switch element {
    case let text as String:
        print("字符串：\(text)")
    case let number as Int:
        print("整数：\(number)")
    case let movie as Movie:
        print("电影：\(movie.title)")
    default:
        print("其他类型")
    }
}

// 2. 协议类型向具体类型转换
protocol Drawable {
    func draw()
}

struct Circle: Drawable {
    func draw() { print("画圆") }
    func area() -> Double { return 3.14 * 5 * 5 }
}

let drawable: Drawable = Circle()
if let circle = drawable as? Circle {
    print("面积：\(circle.area())")
}
```

## 8.4 引用循环与弱引用：weak 和 unowned

引用类型最大的陷阱是**循环引用**。当两个对象互相持有对方的强引用时，双方都无法被释放，造成内存泄漏。

```swift
class Person {
    let name: String
    var pet: Pet?
    
    init(name: String) {
        self.name = name
        print("\(name) 被初始化")
    }
    
    deinit {
        print("\(name) 被释放")
    }
}

class Pet {
    let name: String
    var owner: Person?  // 强引用，导致循环
    
    init(name: String) {
        self.name = name
        print("\(name) 被初始化")
    }
    
    deinit {
        print("\(name) 被释放")
    }
}

var alice: Person? = Person(name: "Alice")
var tom: Pet? = Pet(name: "Tom")

alice?.pet = tom
tom?.owner = alice  // 循环引用！双方都无法被释放

alice = nil  // deinit 不会被调用
tom = nil    // deinit 不会被调用 — 内存泄漏！
```

解决方案是使用 `weak` 或 `unowned` 关键字打破引用循环。

**`weak` 弱引用**：
- 引用对象可能被释放时用 `weak`
- 必须声明为 `var` 和可选类型
- 当被引用对象释放后，自动设置为 `nil`

```swift
class Pet {
    let name: String
    weak var owner: Person?  // 弱引用打破循环
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) 被释放")
    }
}

// 现在双方都可以正确释放
var alice: Person? = Person(name: "Alice")
var tom: Pet? = Pet(name: "Tom")

alice?.pet = tom
tom?.owner = alice

alice = nil  // "Alice 被释放"
// tom.owner 此时已自动变为 nil
tom = nil    // "Tom 被释放"
```

**`unowned` 无主引用**：
- 引用对象生命周期与当前对象相同或更长时使用
- 不声明为可选类型，使用更便捷
- 如果被引用对象提前释放，访问 `unowned` 引用会触发运行时崩溃

```swift
class Customer {
    let name: String
    var card: CreditCard?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) 被释放")
    }
}

class CreditCard {
    let number: String
    unowned let customer: Customer  // 信用卡必然属于某个客户
    
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    
    deinit {
        print("信用卡 \(number) 被释放")
    }
}

var customer: Customer? = Customer(name: "Alice")
customer?.card = CreditCard(number: "1234-5678", customer: customer!)
customer = nil  // 两者都会被释放
```

**闭包中的引用循环**：闭包也是引用类型，当类持有闭包，且闭包捕获了 `self` 时，同样会产生循环引用：

```swift
class NetworkManager {
    var completionHandler: (() -> Void)?
    
    func fetchData() {
        // 闭包捕获了 self，而 self 持有 completionHandler
        completionHandler = {
            print("数据获取完成")
            self.processData()  // 捕获 self
        }
    }
    
    func processData() {
        print("处理数据...")
    }
    
    deinit {
        print("NetworkManager 被释放")
    }
}

var manager: NetworkManager? = NetworkManager()
manager?.fetchData()
manager = nil  // 不会被释放 — 循环引用！
```

解决方式是在闭包的捕获列表中使用 `[weak self]` 或 `[unowned self]`：

```swift
class NetworkManager {
    var completionHandler: (() -> Void)?
    
    func fetchData() {
        completionHandler = { [weak self] in
            guard let self = self else { return }
            print("数据获取完成")
            self.processData()
        }
    }
    
    func processData() {
        print("处理数据...")
    }
    
    deinit {
        print("NetworkManager 被释放")
    }
}

var manager: NetworkManager? = NetworkManager()
manager?.fetchData()
manager = nil  // "NetworkManager 被释放"
```

## 8.5 何时选择类：共享状态、单例、与 Objective-C 互操作

在充分理解引用语义的利弊后，我们总结使用类的典型场景。

**场景一：需要共享状态**

当多个部分需要观察和修改同一份数据时，类是合适的选择：

```swift
class GameState {
    var score: Int = 0 {
        didSet {
            scoreDidUpdate?(score)
        }
    }
    var level: Int = 1 {
        didSet {
            levelDidUpdate?(level)
        }
    }
    
    var scoreDidUpdate: ((Int) -> Void)?
    var levelDidUpdate: ((Int) -> Void)?
}

// 游戏的不同组件共享同一个状态对象
let gameState = GameState()
gameState.scoreDidUpdate = { score in
    print("UI 更新：显示分数 \(score)")
}
gameState.score = 100  // 触发 UI 更新
```

**场景二：单例模式**

某些资源在整个应用生命周期中只需要一个实例，例如日志系统、数据库管理器、UserDefaults 等：

```swift
class Logger {
    // 静态属性保证全局唯一
    static let shared = Logger()
    
    // 私有构造器防止外部创建新实例
    private init() {
        // 初始化日志系统
    }
    
    func log(_ message: String, file: String = #file, line: Int = #line) {
        let filename = (file as NSString).lastPathComponent
        print("[\(filename):\(line)] \(message)")
    }
}

// 全局使用
Logger.shared.log("应用启动")
```

**场景三：与 Objective-C 互操作**

Swift 的许多框架（UIKit、AppKit、Foundation）都是基于 Objective-C 的类体系构建的：

```swift
import UIKit

// 继承 UIKit 类必须使用 class
class MyViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .white
        
        let label = UILabel()
        label.text = "Hello, Swift!"
        label.sizeToFit()
        view.addSubview(label)
    }
}

// 使用 @objc 标记需要暴露给 Objective-C 的方法
@objc class Calculator: NSObject {
    @objc func add(_ a: Int, to b: Int) -> Int {
        return a + b
    }
}
```

**场景四：需要 === 身份判断**

当需要追踪对象的唯一身份，而非其内容值时：

```swift
class NotificationToken {
    let id = UUID()
    // 通知令牌只需要身份，不需要值比较
}
```

**总结**：结构体 vs 类的选择原则

| 考量维度 | 结构体 | 类 |
|---------|-------|-----|
| 值语义 | ✅ 天然支持 | ❌ 引用语义 |
| 继承 | ❌ 不支持 | ✅ 支持 |
| 与 ObjC 互操作 | ❌ 有限 | ✅ 完整 |
| 共享可变状态 | ❌ 不安全 | ✅ 适合 |
| 性能（堆 vs 栈） | ✅ 栈上分配 | ❌ 堆分配+引用计数 |
| 身份判断 | ❌ 无 | ✅ === 运算符 |

当你确信需要引用语义时选择类，其他情况下优先选择结构体。这不是偏见，而是 Swift 语言设计者希望我们遵循的原则——标准库中 `String`、`Array`、`Dictionary` 全部设计为结构体，已经充分表明了立场。
