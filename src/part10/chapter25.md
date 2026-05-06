# 第25章 状态管理与数据流

## 25.1 SwiftUI 单向数据流原则

SwiftUI 数据流的核心理念是**单向数据流 (Unidirectional Data Flow)**。数据从源头出发，沿着视图树向下传递，而事件和用户交互则向上传递，最终回到数据源头形成闭环。

```
  数据源头 (@State, @StateObject, 等)
         │
         ▼  (数据向下流动)
     视图树
         │
         ▼  (事件向上传递)
     用户交互 / 回调
         │
         └──────→ 修改数据源头
```

这一模式带来了几个关键优势：

- **可预测性**：数据总是从父视图流向子视图，子视图不能直接修改父视图的数据。
- **可调试性**：数据变化有明确来源，便于追踪 Bug。
- **性能优势**：SwiftUI 可以精确知道哪个状态发生了变化，只刷新受影响的视图。

### 状态管理全景图

SwiftUI 提供了一系列属性包装器 (Property Wrapper) 来管理不同场景的状态：

| 属性包装器 | 用途 | 值/引用类型 |
|-----------|------|------------|
| `@State` | 视图局部可变状态 | 值类型 |
| `@Binding` | 共享读写权限 | 引用语义 |
| `@StateObject` | 视图拥有引用类型数据 | ObservableObject |
| `@ObservedObject` | 引用外部引用类型数据 | ObservableObject |
| `@EnvironmentObject` | 从环境中隐式获取 | ObservableObject |
| `@Environment` | 读取系统环境值 | 只读 |
| `@AppStorage` | UserDefaults 轻量持久化 | 值类型 |
| `@SceneStorage` | 场景状态恢复 | 值类型 |

## 25.2 @State：局部状态的源头

`@State` 是 SwiftUI 中最基础的状态管理工具，用于管理**属于单个视图的简单值类型状态**。

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("计数: \(count)")
                .font(.largeTitle)
            Button("增加") {
                count += 1  // 直接修改，SwiftUI 自动刷新
            }
        }
    }
}
```

### @State 的工作原理

- `@State` 实际将值存储在视图之外的一块独立堆内存中。
- 即使视图结构体被重新创建，`@State` 的值依然保持。
- 当 `@State` 的值发生变化时，SwiftUI 标记该视图为"脏"状态，并在下一帧重新渲染。

### 使用规则

1. **始终声明为 `private`**：`@State` 属于视图内部状态，不应暴露给外部。
2. **仅用于值类型**：`@State` 应存储值类型（struct、enum、String、Int 等），而非引用类型。
3. **初始化时机**：在视图中直接赋予初始值，不要在 `init` 中设置。

```swift
// ✅ 正确
@State private var isVisible = true
@State private var items = [1, 2, 3]

// ❌ 错误：不要在 init 中赋值
@State private var value: Int
init(value: Int) {
    self._value = State(initialValue: value)  // 仅在需要动态初始值时使用
}
```

## 25.3 @Binding：共享读写权

`@Binding` 提供了一种**不拥有数据但可以读写数据**的机制。它允许子视图修改父视图的状态，而不打破单向数据流。

```swift
struct ParentView: View {
    @State private var isOn = false

    var body: some View {
        ToggleView(isOn: $isOn)  // 传入绑定
    }
}

struct ToggleView: View {
    @Binding var isOn: Bool  // 不拥有数据，只是共享读写

    var body: some View {
        Toggle("开关", isOn: $isOn)
    }
}
```

### 绑定的语法

- 创建绑定：`$stateProperty`（在 `@State` 变量前加 `$` 前缀）。
- 声明绑定：`@Binding var name: Type`。
- 传递绑定：`ChildView(value: $parentState)`。

### 计算绑定

我们也可以创建自定义的 `Binding` 对象：

```swift
struct CustomSlider: View {
    let value: Binding<Double>

    var body: some View {
        Slider(
            value: value,
            in: 0...1
        )
    }
}

// 创建计算 binding
$0.5: Binding<Double> // 字面量语法
```

```swift
let binding = Binding(
    get: { /* 计算当前值 */ },
    set: { newValue in /* 处理新值 */ }
)
```

计算绑定在需要拦截或转换值的读写时非常有用。

## 25.4 @StateObject 与 @ObservedObject：引用类型的状态

当状态需要跨多个视图共享，或包含复杂逻辑时，可以使用符合 `ObservableObject` 协议的引用类型。

### ObservableObject 协议

```swift
class UserSettings: ObservableObject {
    @Published var username = "Guest"
    @Published var isLoggedIn = false
    @Published var preferences: [String: Any] = [:]
}
```

- `@Published`：标记的属性变化时会自动通知观察者。
- `objectWillChange`：ObservableObject 自动合成的发布者，可在属性变化前手动触发通知。

### @StateObject vs @ObservedObject

**`@StateObject`**：视图拥有该对象的生命周期。

```swift
struct ProfileView: View {
    @StateObject private var settings = UserSettings()

    var body: some View {
        Text("Hello, \(settings.username)")
    }
}
```

- 视图创建对象并管理其生命周期。
- 当视图被重新创建时，`@StateObject` 对象**不会被重新创建**——这与 `@State` 行为类似。
- 应使用 `@StateObject` 作为**数据源 (source of truth)**。

**`@ObservedObject`**：视图引用由外部传入的对象。

```swift
struct ProfileView: View {
    @ObservedObject var settings: UserSettings  // 由父视图传入

    var body: some View {
        Text("Hello, \(settings.username)")
    }
}
```

- 视图不拥有生命周期，对象由父视图或外部持有。
- 父视图应使用 `@StateObject` 创建对象，通过初始化或 `@EnvironmentObject` 传给子视图。
- ⚠️ 不要同时使用 `@ObservedObject` 初始化对象。

### 关键区别

| 特性 | @StateObject | @ObservedObject |
|------|-------------|----------------|
| 生命周期 | 由视图管理 | 由外部管理 |
| 对象创建 | 在视图中创建 | 外部传入 |
| 视图重建 | 对象保持 | 对象可能丢失 |
| 推荐场景 | 数据源 | 引用已有数据 |

## 25.5 @EnvironmentObject：跨层级注入

当数据需要在视图树中**多层级传递**时，逐层使用初始化参数传递非常繁琐。`@EnvironmentObject` 解决了这个问题。

### 注入与使用

**步骤 1：在根视图注入**
```swift
@main
struct MyApp: App {
    @StateObject private var settings = UserSettings()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(settings)  // 注入到环境
        }
    }
}
```

**步骤 2：在任何子视图中获取**
```swift
struct SomeDeepView: View {
    @EnvironmentObject var settings: UserSettings  // 从环境隐式获取

    var body: some View {
        Text("Username: \(settings.username)")
    }
}
```

### 使用规则

1. 向环境中注入的对象类型**必须是唯一的**——每个类型只能注入一个实例。
2. 如果某个子视图获取了环境中不存在的类型，**运行时将引发 Crash**。
3. 适合全局共享的数据，如用户设置、主题、认证状态等，不适合局部的、临时的数据。

### 何时使用

| 场景 | 推荐方式 |
|------|---------|
| 数据仅在一两个层级传递 | 构造参数 |
| 数据在多个分支深层传递 | @EnvironmentObject |
| 系统提供的全局值 | @Environment |

## 25.6 @Environment：读取系统环境值

`@Environment` 用于读取 SwiftUI 系统提供的环境值，这些值描述了当前运行环境的上下文：

```swift
struct DetailView: View {
    @Environment(\.colorScheme) var colorScheme
    @Environment(\.locale) var locale
    @Environment(\.dismiss) var dismiss
    @Environment(\.horizontalSizeClass) var horizontalSizeClass

    var body: some View {
        VStack {
            Text("当前模式: \(colorScheme == .dark ? "深色" : "浅色")")
            Text("语言区域: \(locale.identifier)")

            if horizontalSizeClass == .compact {
                Text("紧凑布局")
            }

            Button("关闭") {
                dismiss()
            }
        }
    }
}
```

### 自定义环境值

你可以通过自定义 `EnvironmentKey` 向环境中添加自己的值：

```swift
struct ThemeKey: EnvironmentKey {
    static let defaultValue: Theme = .system
}

extension EnvironmentValues {
    var theme: Theme {
        get { self[ThemeKey.self] }
        set { self[ThemeKey.self] = newValue }
    }
}

// 注入自定义环境值
ContentView()
    .environment(\.theme, .dark)
```

## 25.7 @AppStorage 与 @SceneStorage：轻量持久化

### @AppStorage：UserDefaults 持久化

`@AppStorage` 是对 `UserDefaults` 的 SwiftUI 原生封装，适合存储轻量设置：

```swift
struct SettingsView: View {
    @AppStorage("username") var username = "Guest"
    @AppStorage("fontSize") var fontSize = 16
    @AppStorage("isDarkMode") var isDarkMode = false

    var body: some View {
        Form {
            TextField("用户名", text: $username)
            Stepper("字号: \(fontSize)", value: $fontSize, in: 12...24)
            Toggle("深色模式", isOn: $isDarkMode)
        }
    }
}
```

使用注意：

- 键名建议使用常量或枚举管理，避免拼写错误。
- 支持的类型有限：String、Int、Double、Bool、Data、URL 及 RawRepresentable 类型。
- 适合存储少量配置数据，不适合存储大量数据或敏感信息。

### @SceneStorage：场景状态恢复

`@SceneStorage` 与 `@AppStorage` 类似，但数据仅在当前场景 (Scene) 的生命周期内有效，常用于多窗口场景的状态保持：

```swift
struct DocumentView: View {
    @SceneStorage("selectedTab") var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            Text("标签 1").tag(0)
            Text("标签 2").tag(1)
            Text("标签 3").tag(2)
        }
    }
}
```

在 iPadOS 或 macOS 的多窗口环境中，每个窗口有自己的 `@SceneStorage` 空间。

## 25.8 自定义 Binding 与高级数据流模式

### 自定义 Binding 的场景

当我们需要在状态读写之间添加转换逻辑时，自定义 Binding 非常有用：

```swift
struct TemperatureView: View {
    @State private var celsius: Double = 0

    var body: some View {
        let fahrenheit = Binding<Double>(
            get: { celsius * 9 / 5 + 32 },
            set: { celsius = ($0 - 32) * 5 / 9 }
        )

        VStack {
            Slider(value: fahrenheit, in: 32...212)
            Text("摄氏: \(celsius, specifier: "%.1f")°C")
            Text("华氏: \(fahrenheit.wrappedValue, specifier: "%.1f")°F")
        }
    }
}
```

### Binding 的变换方法

SwiftUI 为 Binding 提供了一些内置变换方法：

```swift
// 可选值绑定
@State var name: String?
TextField("Name", text: Binding($name) ?? "")

// 映射到可选值
TextField("Name", text: $name ?? "")

// 映射变换
let optionalBinding: Binding<String?> = // ...
let stringBinding = optionalBinding ?? ""
```

### 高级模式：中央状态管理

对于大型应用，可以将状态管理集中到一个 Store 中：

```swift
class AppStore: ObservableObject {
    @Published var user: User?
    @Published var settings: AppSettings
    @Published var tasks: [Task] = []

    func addTask(_ task: Task) { /* ... */ }
    func removeTask(_ id: UUID) { /* ... */ }
}

// 在根注入
@main
struct MyApp: App {
    @StateObject private var store = AppStore()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(store)
        }
    }
}

// 在视图中使用
struct TaskListView: View {
    @EnvironmentObject var store: AppStore

    var body: some View {
        List {
            ForEach(store.tasks) { task in
                TaskRow(task: task)
            }
        }
    }
}
```

这种模式结合了 `@StateObject` 和 `@EnvironmentObject` 的优势，既能集中管理状态，又能避免繁琐的逐层传递。

### 选择合适的状态管理策略

选择何种状态管理方式，取决于数据的**作用域**和**生命周期**：

- **视图私有状态**（如输入框文本、开关状态）→ `@State`
- **需要在子视图共享** → `@Binding`
- **复杂业务逻辑，局部数据源** → `@StateObject`
- **跨视图共享，从父视图传入** → `@ObservedObject` + 初始化参数
- **全局共享，贯穿整个应用** → `@StateObject` + `@EnvironmentObject`
- **系统环境值** → `@Environment`
- **轻量持久化** → `@AppStorage`
- **场景恢复** → `@SceneStorage`

## 总结

本章深入探讨了 SwiftUI 的数据流体系。我们从单向数据流原则出发，依次学习了 @State 管理局部状态、@Binding 实现共享读写、@StateObject 与 @ObservedObject 处理引用类型状态、@EnvironmentObject 跨层级注入、@Environment 读取系统环境值、@AppStorage 与 @SceneStorage 的轻量持久化方案，以及自定义 Binding 和高级数据流模式。掌握这些状态管理工具，是构建从简单到复杂的 SwiftUI 应用的基础。
