# 第19章 互操作性：与 Objective-C 共存

## 19.1 桥接原理：NSString 与 String

Swift 与 Objective-C 的互操作性建立在**无缝桥接（Toll-Free Bridging）** 的基础之上。在 Core Foundation 层面，`NSString` 和 `String` 实际上是同一对象——`NSString` 可以直接当作 `String` 使用，反之亦然。

### 桥接透明性

```swift
import Foundation

// NSString 到 String 的隐式桥接
let nsString: NSString = "Hello, Objective-C"
let swiftString: String = nsString as String
print(swiftString.count) // Swift 的 String API

// String 到 NSString 的隐式桥接
let greeting: String = "Hello, Swift"
let objcString: NSString = greeting as NSString
print(objcString.length) // NSString 的 length 属性
```

这种桥接是零开销的——不需要拷贝字符数据。当 Swift `String` 被桥接为 `NSString` 时，只是类型标记的转换，底层字符缓冲区保持一致。

### 桥接的性能特征

虽然桥接本身没有数据拷贝开销，但 `String` 与 `NSString` 的 API 差异可能导致隐式转换成本：

- `NSString.length` 返回 UTF-16 编码的码元数量。
- `String.count` 返回 Unicode 扩展字素簇（Extended Grapheme Clusters）的数量。

这两种计数方式在字符串包含复合字符（如 emoji）时结果不同。因此，在 Swift 中频繁访问 `.count` 而底层是 `NSString` 时，可能会触发重算。

### 集合类型的桥接

```swift
let nsArray: NSArray = [1, 2, 3]
let swiftArray: [Int] = nsArray as! [Int] // 桥接为 Swift Array

let nsDictionary: NSDictionary = ["a": 1, "b": 2]
let swiftDict: [String: Int] = nsDictionary as! [String: Int]
```

与字符串类似，`NSArray` ↔ `[Any]`、`NSDictionary` ↔ `[AnyHashable: Any]` 等桥接也是零开销的。但 Element 类型为值类型时，从 `NSArray` 中取出元素会触发拷贝。

## 19.2 可空性注解与可选

Objective-C 中对象指针可以为 `nil`，但在 Swift 引入之前，类型系统并不区分可空与非空。为了解决这一不安全性，Apple 引入了 **可空性注解（Nullability Annotations）**。

### _Nullable 与 _Nonnull

```objc
// Objective-C 头文件
@interface UserService : NSObject
- (NSString * _Nonnull)fetchUserName;       // 不会返回 nil
- (NSString * _Nullable)fetchOptionalName;  // 可能返回 nil
- (NSString *)fetchUnannotatedName;         // 未注解，被视为隐式解包可选
@end
```

在 Swift 中，上述声明分别对应：

```swift
// Swift 中对应的桥接
class UserService: NSObject {
    func fetchUserName() -> String          // 非可选
    func fetchOptionalName() -> String?     // 可选
    func fetchUnannotatedName() -> String!  // 隐式解包可选（IUO）
}
```

### NS_ASSUME_NONNULL_BEGIN/END

为减少注解的重复编写，Objective-C 提供了区域宏：

```objc
NS_ASSUME_NONNULL_BEGIN

@interface UserService : NSObject
- (NSString *)fetchUserName;         // 默认 _Nonnull
- (nullable NSString *)fetchOptionalName;
@property (nonatomic, copy, nullable) NSString *nickname;
@end

NS_ASSUME_NONNULL_END
```

在 Swift 5.x 中，从 Objective-C 导入的未经注解的指针类型会被视为 `Optional`，而非早期版本中的 `ImplicitlyUnwrappedOptional`。这提升了类型安全性，但可能导致编译器警告，需要在 Objective-C 侧补全可空性注解。

## 19.3 动态派发与 @objc

`@objc` 属性用于将 Swift 声明暴露给 Objective-C 运行时。它是互操作性的关键机制。

### 使用 @objc 暴露方法

```swift
class MyViewController: UIViewController {
    
    @objc func handleTap() {
        print("按钮被点击")
    }
    
    @objc private func internalAction() {
        // private 方法也可通过 @objc 暴露给 OC 运行时
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        let button = UIButton()
        button.addTarget(self, action: #selector(handleTap), for: .touchUpInside)
    }
}
```

`@objc` 的代价是函数调用变为动态派发——通过 Objective-C 的 `objc_msgSend` 机制。对于高频调用的方法，应避免滥用 `@objc`。

### @objcMembers

如果一个类的大部分成员都需要暴露给 Objective-C，可以在类上使用 `@objcMembers`：

```swift
@objcMembers class AnalyticsEvent: NSObject {
    var name: String
    var value: Int
    var timestamp: Date
    
    func log() { /* ... */ }
}
// 以上所有属性和方法自动隐式添加 @objc
```

`@objcMembers` 简化了书写，但也会扩大二进制文件大小，因为编译器需为每个成员生成 Objective-C 元数据。

### 动态替换与 KVO

`@objc dynamic` 组合使 Swift 属性具备键值观察（KVO）和动态替换（Method Swizzling）的能力：

```swift
class ObservableObject: NSObject {
    @objc dynamic var value: String = ""
}

let obj = ObservableObject()
obj.observe(\.value, options: [.new]) { object, change in
    print("值变为: \(change.newValue ?? "")")
}
obj.value = "新值" // 触发观察回调
```

`dynamic` 修饰符强制通过 Objective-C 运行时动态派发，而不是 Swift vtable。这是互操作性场景下的必要代价。

## 19.4 Swift 中使用 C/ObjC 库

### 使用 Objective-C 库

只要 UIKit、Foundation 等系统框架本身就是基于 Objective-C 实现的，在 Swift 中可以直接导入使用。对于第三方 Objective-C 库，通过桥接头文件（Bridging Header）导入：

```objc
// BridgingHeader.h
#import "SomeObjCLibrary.h"
```

然后在 Swift 中直接使用：

```swift
// Swift 文件
let object = SomeObjCClass()
object.doSomething(with: "参数")
```

### 使用 C 函数与类型

Swift 可以导入 C 头文件中声明的函数、枚举、结构体和宏（部分）。

```c
// C 头文件
typedef struct {
    double x;
    double y;
} Point2D;

double distance(Point2D a, Point2D b);
```

```swift
// Swift 中使用
let p1 = Point2D(x: 0, y: 0)
let p2 = Point2D(x: 3, y: 4)
let dist = distance(p1, p2) // 5.0
```

### C 指针的处理

C 函数中的指针参数在 Swift 中映射为 `UnsafePointer` 或 `UnsafeMutablePointer`：

```swift
// C: void update(int *value);
func update(_ value: UnsafeMutablePointer<Int32>!)

var x: Int32 = 10
update(&x)
```

### C 宏与复杂构造

Swift 无法直接导入复杂的 C 宏（如 `#define MIN(a,b) ((a)<(b)?(a):(b))`）。这些宏需要在 Objective-C 中封装为函数，再通过桥接头暴露给 Swift。

## 19.5 何时不再需要 Objective-C

### Objective-C 依赖的减少

随着 Swift 生态系统的成熟，以下场景已不再需要 Objective-C：

#### 1. 使用 Swift 原生特性替代 Cocoa 模式

```swift
// 替代 KVO：使用 Swift 的观察者或 Combine
class Monitor {
    @Published var state: State = .idle
    
    var cancellables: Set<AnyCancellable> = []
    func start() {
        $state.sink { newState in
            print("状态更新: \(newState)")
        }.store(in: &cancellables)
    }
}
```

#### 2. 纯 Swift 库与框架

越来越多的库以纯 Swift 形式发布，不再依赖 Objective-C 运行时。例如 SwiftUI、Alamofire、SwiftNIO 等框架完全不需要桥接头文件。

#### 3. 新项目优先纯 Swift

对于全新的应用程序，使用纯 Swift 已成为推荐实践。仅以下场景仍需要 Objective-C：

- 维护现有的 Objective-C 代码库。
- 使用 KVO 或 Method Swizzling 等动态特性（可通过 `@objc dynamic` 解决）。
- 依赖尚未提供 Swift 版本的第三方库。
- 需要 `performSelector:` 等运行时消息发送。

### 未来的方向

Apple 持续推动从 Objective-C 到 Swift 的迁移：

- **SwiftUI** 完全基于 Swift，替代 UIKit。
- **Swift Concurrency** 提供原生的 async/await 和 Actor 模型。
- **Swift Macro** 替代 Objective-C 预处理器宏。

Objective-C 不会很快消失——庞大的 Cocoa 生态系统和运行时灵活性使其依然有存在价值。但对于新代码，选择 Swift 意味着更安全的类型系统、更好的性能优化（得益于 SIL）和更现代化的语言特性。

## 小结

Swift 与 Objective-C 的互操作性是 Swift 成功的关键因素之一。通过无缝桥接、可空性注解和 `@objc` 动态派发，Swift 得以在利用已有 Cocoa 生态的同时逐步发展自己的语言特性。在新时代的 Swift 开发中，Objective-C 的依赖正逐步减少，理解和掌握这种互操作机制，是编写高质量 Swift 代码的必备技能。
