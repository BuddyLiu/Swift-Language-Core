# 第23章 声明式范式与 View 协议

## 23.1 从命令式到声明式

在 SwiftUI 出现之前，iOS/macOS 开发的主流范式是**命令式编程**。开发者需要通过一系列指令，明确告诉程序 "何时做、做什么、怎么做"。UIKit 就是典型的命令式框架。

### UIKit 的命令式风格

在 UIKit 中，构建一个简单的带标题和按钮的界面通常需要编写大量代码：

```swift
class GreetingViewController: UIViewController {
    private let label = UILabel()
    private let button = UIButton(type: .system)
    private var tapCount = 0

    override func viewDidLoad() {
        super.viewDidLoad()

        // 配置 label
        label.text = "Hello, World!"
        label.textAlignment = .center
        label.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(label)

        // 配置 button
        button.setTitle("Tap me", for: .normal)
        button.translatesAutoresizingMaskIntoConstraints = false
        button.addTarget(self, action: #selector(buttonTapped), for: .touchUpInside)
        view.addSubview(button)

        // 布局约束
        NSLayoutConstraint.activate([
            label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            label.centerYAnchor.constraint(equalTo: view.centerYAnchor, constant: -20),
            button.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            button.topAnchor.constraint(equalTo: label.bottomAnchor, constant: 20)
        ])
    }

    @objc func buttonTapped() {
        tapCount += 1
        label.text = "Tapped \(tapCount) times"
    }
}
```

这段代码详细描述了每一步操作：创建视图对象、配置属性、添加到视图层级、设置约束、添加事件处理。**"如何做" 的细节完全暴露在业务代码中**，使得界面逻辑与实现细节高度耦合。

### SwiftUI 的声明式风格

同样的功能，用 SwiftUI 实现则简洁得多：

```swift
struct GreetingView: View {
    @State private var tapCount = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("Hello, World!")
                .font(.title)

            Text("Tapped \(tapCount) times")
                .foregroundColor(.gray)

            Button("Tap me") {
                tapCount += 1
            }
        }
    }
}
```

关键差异在于：

- **描述 "是什么"**：我们声明界面由 `VStack`、`Text` 和 `Button` 组成，而不是一步步创建它们。
- **自动更新**：当 `tapCount` 变化时，SwiftUI 自动重新计算 `body` 并更新界面，无需手动调用 `label.text = ...`。
- **更少的样板代码**：布局、事件绑定、状态管理全部集成在声明中。

> 声明式编程的核心思想是：**你描述目标状态，框架负责实现从当前状态到目标状态的转变。**

## 23.2 View 协议：一切皆为视图

在 SwiftUI 中，**一切皆为视图 (View)**。无论是一个文本标签、一个按钮、一个图片，还是一个容器，所有界面元素都遵循 `View` 协议。

### View 协议的定义

```swift
public protocol View {
    associatedtype Body: View
    @ViewBuilder var body: Body { get }
}
```

协议只有两个核心要求：

1. **`Body` 关联类型**：表示 `body` 返回的具体视图类型。由于 SwiftUI 大量使用泛型和 opaque type，绝大多数情况下我们无需显式指定，只需写 `some View`。
2. **`body` 计算属性**：视图的内容由 `body` 属性定义。SwiftUI 会在需要时调用 `body` 来获取当前视图的描述。

### 空视图：`EmptyView` 与 `Never`

某些特殊类型也遵循 `View` 协议：

- `EmptyView`：表示一个空视图，常用于条件分支中不展示任何内容的情况。
- `Never`：某些视图的 `Body` 类型为 `Never`（如 `Text`、`Image`），表示它们是不可再分的基本视图——它们是视图树的叶子节点。

### 结构体视图的 Value Semantics

SwiftUI 中的视图通常被定义为结构体（struct），这意味着它们具有**值语义**：

```swift
struct CustomView: View {
    let title: String
    let color: Color

    var body: some View {
        Text(title)
            .foregroundColor(color)
    }
}
```

结构体的不可变性帮助 SwiftUI 高效地比较视图树：当状态变化时，SwiftUI 通过值比较快速找出需要更新的视图节点。

## 23.3 body 属性与视图的重新渲染

### body 的计算时机

SwiftUI 在以下时机调用视图的 `body` 属性：

1. **视图首次出现在屏幕上**时。
2. **依赖的状态或数据发生变化**时（如 `@State`、`@ObservedObject` 等属性包装器标识的数据源发生变化）。
3. **父视图的布局或尺寸发生变化**，且该视图依赖于几何信息时。

### 依赖驱动刷新

SwiftUI 的刷新机制是**依赖驱动的**。这意味着只有 `body` 中实际使用了某个状态变量，当该变量变化时，该视图才会被重新渲染。

```swift
struct CounterView: View {
    @State private var count = 0
    @State private var message = "Hello"

    var body: some View {
        VStack {
            Text(message)      // 只依赖 message
            Text("Count: \(count)") // 依赖 count
            Button("Increment") { count += 1 }
        }
    }
}
```

当 `count` 变化时，只有 `Text("Count: ...")` 会重新渲染，`Text(message)` 不会受影响。SwiftUI 通过依赖跟踪系统自动管理这种行为。

### 性能优化：最小化刷新范围

虽然 SwiftUI 自动做了很多优化，但我们仍应注意：

- 将大视图拆分为小视图，让状态变化只触发小范围刷新。
- 使用 `EquatableView` 或 `equatable()` 修饰符为复杂视图提供自定义相等判断。
- 避免在 `body` 中执行昂贵计算，考虑将计算结果缓存。

## 23.4 修饰符链：视图的配置与转换

### 修饰符的本质

修饰符 (Modifier) 是 SwiftUI 中最核心的扩展机制。每个修饰符方法接收一个视图，返回一个经过包装的新视图：

```swift
Text("Hello")
    .font(.title)        // 返回 ModifiedContent<Text, FontModifier>
    .foregroundColor(.blue) // 返回 ModifiedContent<..., ForegroundColorModifier>
    .padding()           // 返回 ModifiedContent<..., PaddingModifier>
    .background(Color.yellow) // 返回 ModifiedContent<..., BackgroundModifier>
```

从类型角度看，每个修饰符都会**改变视图的泛型类型**。这也是为什么 SwiftUI 中使用 `some View` 而非具体的 `Text` 类型——因为我们几乎总是应用了一串修饰符。

### 修饰符的顺序

修饰符的**应用顺序**很重要：

```swift
// 情况 1：先 padding 再 background
Text("Hello")
    .padding(20)
    .background(Color.yellow)
// 结果：黄色背景覆盖了 padding 区域

// 情况 2：先 background 再 padding
Text("Hello")
    .background(Color.yellow)
    .padding(20)
// 结果：文字后有黄色背景，但 padding 区域是透明的
```

理解顺序的关键：每个修饰符创建一个新层，外层修饰符包裹内层结果。

### 自定义修饰符

当一组修饰符被重复使用时，可以通过 `ViewModifier` 协议封装：

```swift
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding(16)
            .background(Color.white)
            .cornerRadius(12)
            .shadow(radius: 4)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardStyle())
    }
}

// 使用
Text("Hello").cardStyle()
```

## 23.5 ViewBuilder 与结果构建器

### @ViewBuilder 的作用

`@ViewBuilder` 是一个**结果构建器 (Result Builder)**，它允许我们在一个闭包中组合多个视图，而无需使用元组或数组包装：

```swift
// 不需要 ViewBuilder 时，只能返回单一视图
var body: some View {
    Text("Hello")
}

// 有了 @ViewBuilder，可以组合多个视图
@ViewBuilder
var body: some View {
    Text("Hello")
    Text("World")
    Divider()
    Text("SwiftUI")
}
```

### @resultBuilder 原理

Swift 5.1 引入 `@resultBuilder` 特性，让我们可以自定义结果构建器。`@ViewBuilder` 本质上是标准库中定义的一个结果构建器：

```swift
@resultBuilder
struct ViewBuilder {
    // 处理空闭包
    static func buildBlock() -> EmptyView

    // 处理单个视图
    static func buildBlock<Content: View>(_ content: Content) -> Content

    // 处理两个视图 → TupleView
    static func buildBlock<C0, C1>(_ c0: C0, _ c1: C1) -> TupleView<(C0, C1)>

    // 处理三到十个视图...
    // buildIf、buildEither、buildOptional 等
}
```

关键方法：

- **`buildBlock`**：将多个视图组合成一个 `TupleView`。最多支持 10 个子视图。
- **`buildIf`**：支持 `if` 条件语句。
- **`buildEither(first:)` / `buildEither(second:)`**：支持 `if/else` 和 `switch` 语句。
- **`buildOptional`**：支持可选视图。
- **`buildLimitedAvailability`**：支持 `#available` 条件。

### @ViewBuilder 支持的控制流

```swift
@ViewBuilder
func profileView(isLoggedIn: Bool, name: String?) -> some View {
    if isLoggedIn {
        Text("Welcome, \(name ?? "User")!")
            .font(.title)
    } else {
        Button("Log In") { /* ... */ }
    }

    if let displayName = name {
        Divider()
        Text("Display name: \(displayName)")
    }
}
```

## 23.6 视图的组合与提取

### 为什么要提取子视图

当 `body` 变得复杂时，提取子视图可以带来多重好处：

1. **可读性**：将大视图拆分为命名清晰的子视图。
2. **复用性**：子视图可以在不同地方重用。
3. **性能**：子视图拥有独立的依赖跟踪范围，某个子视图的状态变化不会触发其他子视图的重新渲染。

### 提取方式一：计算属性

```swift
struct ProfileView: View {
    let user: User

    var body: some View {
        VStack {
            headerView
            userInfoView
            actionButtonsView
        }
    }

    private var headerView: some View {
        HStack {
            AsyncImage(url: user.avatarURL)
                .frame(width: 60, height: 60)
                .clipShape(Circle())
            Text(user.name)
                .font(.headline)
        }
    }

    private var userInfoView: some View {
        Group {
            Text(user.bio)
                .font(.body)
            Text(user.location)
                .font(.caption)
                .foregroundColor(.secondary)
        }
    }

    private var actionButtonsView: some View {
        HStack {
            Button("Follow") { /* ... */ }
            Button("Message") { /* ... */ }
        }
    }
}
```

### 提取方式二：独立结构体

当子视图有自己的逻辑或状态时，应提取为独立的 `View` 结构体：

```swift
struct LikeButton: View {
    @Binding var isLiked: Bool
    let onTap: () -> Void

    var body: some View {
        Button(action: onTap) {
            Image(systemName: isLiked ? "heart.fill" : "heart")
                .foregroundColor(isLiked ? .red : .gray)
        }
    }
}

// 在父视图中使用
struct PostView: View {
    @State private var isLiked = false

    var body: some View {
        LikeButton(isLiked: $isLiked) {
            print("Toggle like")
        }
    }
}
```

### @ViewBuilder 与条件视图

有时我们需要在父视图中动态决定显示哪些子视图，这时可以利用 `@ViewBuilder` 的函数参数：

```swift
struct CardView<Content: View>: View {
    let title: String
    @ViewBuilder let content: Content

    var body: some View {
        VStack(alignment: .leading) {
            Text(title)
                .font(.headline)
            content
        }
        .padding()
        .background(Color.gray.opacity(0.1))
        .cornerRadius(8)
    }
}

// 使用
CardView(title: "Settings") {
    Text("Option 1")
    Toggle("Enable", isOn: $isEnabled)
    Slider(value: $value)
}
```

这种模式让 API 变得极其灵活——调用者可以传入任意数量的子视图，而 `CardView` 只需通过 `@ViewBuilder` 接收并展示即可。

## 总结

本章介绍了 SwiftUI 声明式范式的核心理念与 View 协议。我们从 UIKit 与 SwiftUI 的对比出发，理解了声明式编程的优越性；深入剖析了 View 协议的定义、body 属性的计算机制；掌握了修饰符链的工作原理与自定义修饰符；学习了 @ViewBuilder 的内部实现机制；最后探讨了视图组合与提取的最佳实践。这些知识是掌握 SwiftUI 的基础，后续章节将在其上构建更复杂的布局与数据流体系。
