# 第26章 导航与模态展示

在 SwiftUI 中，导航和模态展示是构建多页面应用程序的核心能力。随着 SwiftUI 的持续演进，Apple 在 iOS 16/macOS 13 中引入了全新的导航体系，彻底替代了旧的 `NavigationView`。本章将全面覆盖现代 SwiftUI 导航技术栈，包括 NavigationStack、NavigationSplitView、TabView，以及 Sheet 和 Popover 等模态展示方式，最后介绍编程式导航与深层链接的实现。

## 26.1 NavigationStack 与导航路径（NavigationPath）

`NavigationStack` 是 iOS 16+ 中引入的导航容器，取代了旧的 `NavigationView`。它基于**值驱动**的导航模型，即通过数据类型的压栈和出栈来控制页面的转换。

### 基本用法

最简单的 `NavigationStack` 使用 `NavigationLink` 实现目标页面的跳转：

```swift
struct ContentView: View {
    var body: some View {
        NavigationStack {
            List(1..<20) { i in
                NavigationLink("第 \(i) 项", value: i)
            }
            .navigationDestination(for: Int.self) { value in
                DetailView(item: value)
            }
            .navigationTitle("列表")
        }
    }
}

struct DetailView: View {
    let item: Int
    var body: some View {
        Text("你选择了第 \(item) 项")
            .font(.title)
            .navigationTitle("详情")
    }
}
```

关键区别在于，`navigationDestination(for:destination:)` 根据数据类型注册目标视图，而非直接将目标视图嵌套在 `NavigationLink` 中。这种分离使得导航逻辑更加清晰，也更容易支持深层链接。

### NavigationPath

当需要管理复杂的导航路径（如多级跳转或动态路由）时，可以使用 `NavigationPath`：

```swift
struct PathExample: View {
    @State private var navPath = NavigationPath()

    var body: some View {
        NavigationStack(path: $navPath) {
            VStack(spacing: 20) {
                Button("跳转到 A 页面") {
                    navPath.append("PageA")
                }
                Button("跳转到 B 页面（带ID）") {
                    navPath.append(100)
                }
                Button("跳转多级") {
                    navPath.append("PageA")
                    navPath.append(200)
                }
            }
            .navigationDestination(for: String.self) { value in
                Text("字符串页面: \(value)")
            }
            .navigationDestination(for: Int.self) { value in
                Text("整数页面: \(value)")
            }
            .navigationTitle("导航路径")
        }
    }
}
```

`NavigationPath` 是一个类型擦除的路径容器，可以存储异构类型的数据。每个入栈的数据项对应一次导航跳转，当需要返回时，可以直接修改 `navPath`：

```swift
// 返回到根视图
navPath.removeLast(navPath.count)

// 或返回到指定层级
navPath.removeLast(2)
```

> 注意：`NavigationPath` 要求入栈的数据类型遵循 `Hashable` 协议。

### 与旧版 NavigationView 的对比

| 特性 | NavigationView (已废弃) | NavigationStack |
|------|------------------------|-----------------|
| 导航模型 | 视图嵌套驱动 | 值类型驱动 |
| 编程式导航 | 通过 `isActive` 绑定 | 通过 `path` 绑定 |
| 类型安全 | 否 | 是 |
| 深层链接支持 | 困难 | 原生支持 |
| 性能 | 较差（预先加载所有目标视图） | 优化（延迟加载） |

## 26.2 NavigationSplitView：多栏布局（iPadOS/macOS）

`NavigationSplitView` 是 SwiftUI 中实现分栏导航（如邮件应用的侧边栏 + 内容区）的专用视图，特别适合 iPadOS 和 macOS 的大屏幕场景。

### 两栏布局

```swift
struct TwoColumnLayout: View {
    @State private var selectedCategory: String?
    @State private var selectedItem: String?

    var body: some View {
        NavigationSplitView {
            // 侧边栏
            List(Category.samples, id: \.name, selection: $selectedCategory) { category in
                Text(category.name)
                    .font(.headline)
            }
            .navigationTitle("分类")
        } detail: {
            // 内容区
            if let category = selectedCategory {
                List(Category.items(for: category), id: \.self, selection: $selectedItem) { item in
                    Text(item)
                }
                .navigationTitle(category)
            } else {
                Text("请选择一个分类")
                    .foregroundColor(.secondary)
            }
        }
    }
}
```

### 三栏布局

在邮件或文件管理类应用中，三栏布局更常见：

```swift
struct ThreeColumnLayout: View {
    @State private var selectedCategory: String?
    @State private var selectedItem: String?
    @State private var selectedDetail: String?

    var body: some View {
        NavigationSplitView {
            // 第一栏：分类
            List(Category.samples, id: \.name, selection: $selectedCategory) { category in
                Text(category.name)
            }
            .navigationTitle("分类")
        } content: {
            // 第二栏：列表
            List(items, id: \.self, selection: $selectedDetail) { detail in
                Text(detail)
            }
            .navigationTitle(selectedCategory ?? "")
        } detail: {
            // 第三栏：详情
            if let detail = selectedDetail {
                DetailContentView(text: detail)
            } else {
                Text("选择查看详情")
            }
        }
    }
}
```

`NavigationSplitView` 会自动适配屏幕宽度：在紧凑宽度下折叠为单栏导航，在常规宽度下显示多栏视图。

### 控制栏的可见性

```swift
NavigationSplitView(columnVisibility: $columnVisibility) {
    // 侧边栏
} content: {
    // 内容栏
} detail: {
    // 详情栏
}
```

`columnVisibility` 的类型为 `NavigationSplitViewVisibility`，可选值包括 `.all`、`.doubleColumn`、`.detailOnly` 和 `.automatic`。

## 26.3 TabView 与分页

`TabView` 是 iOS 应用中最常用的顶级导航容器，提供底部标签栏切换不同功能模块。

### 标准标签栏

```swift
struct MainTabView: View {
    @State private var selectedTab = 0

    var body: some View {
        TabView(selection: $selectedTab) {
            HomeView()
                .tabItem {
                    Label("首页", systemImage: "house")
                }
                .tag(0)

            SearchView()
                .tabItem {
                    Label("搜索", systemImage: "magnifyingglass")
                }
                .tag(1)

            SettingsView()
                .tabItem {
                    Label("设置", systemImage: "gear")
                }
                .tag(2)
        }
        .onChange(of: selectedTab) { oldValue, newValue in
            print("切换到标签: \(newValue)")
        }
    }
}
```

### 分页样式

在 watchOS 或需要滑动切换的场景中，可以使用分页样式：

```swift
struct PagingTabView: View {
    var body: some View {
        TabView {
            PageView(color: .red, text: "第一页")
            PageView(color: .green, text: "第二页")
            PageView(color: .blue, text: "第三页")
        }
        .tabViewStyle(.page)
        .indexViewStyle(.page(backgroundDisplayMode: .always))
    }
}

struct PageView: View {
    let color: Color
    let text: String

    var body: some View {
        ZStack {
            color.ignoresSafeArea()
            Text(text)
                .font(.largeTitle)
                .foregroundColor(.white)
        }
    }
}
```

### 自定义标签栏外观

通过 `UITabBarAppearance` 可以自定义标签栏的外观：

```swift
init() {
    let appearance = UITabBarAppearance()
    appearance.configureWithOpaqueBackground()
    appearance.backgroundColor = UIColor.systemBackground
    UITabBar.appearance().standardAppearance = appearance
    UITabBar.appearance().scrollEdgeAppearance = appearance
}
```

## 26.4 Sheet、Popover、FullScreenCover

SwiftUI 提供了多种模态展示方式以满足不同的交互场景。

### Sheet

`sheet` 是最常用的模态展示方式，从底部滑入一个视图：

```swift
struct SheetExample: View {
    @State private var showSheet = false

    var body: some View {
        Button("显示 Sheet") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            SheetContentView()
        }
    }
}

struct SheetContentView: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Text("Sheet 内容")
                .toolbar {
                    ToolbarItem(placement: .confirmationAction) {
                        Button("完成") { dismiss() }
                    }
                    ToolbarItem(placement: .cancellationAction) {
                        Button("取消") { dismiss() }
                    }
                }
        }
    }
}
```

#### 自定义 Sheet 尺寸（iOS 16+）

```swift
.sheet(isPresented: $showSheet) {
    SheetContentView()
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
        .presentationBackground(.regularMaterial)
}
```

- `presentationDetents`：控制 Sheet 的高度范围（`.medium`、`.large`、`.fraction(0.3)`、`.height(300)`）
- `presentationDragIndicator`：是否显示拖拽手柄
- `presentationBackground`：设置背景样式
- `presentationCornerRadius`：设置圆角半径（iOS 16.4+）

### Popover

Popover 在 iPadOS 上以浮动气泡的形式展示，在 iOS 上自动回退为 Sheet：

```swift
struct PopoverExample: View {
    @State private var showPopover = false

    var body: some View {
        Button("显示 Popover") {
            showPopover = true
        }
        .popover(isPresented: $showPopover) {
            Text("这是 Popover 内容")
                .padding()
                .presentationCompactAdaptation(.popover)
        }
    }
}
```

`presentationCompactAdaptation` 可以控制 Popover 在紧凑环境下的适配行为。

### FullScreenCover

需要全屏覆盖时（如引导页、登录页），使用 `fullScreenCover`：

```swift
.fullScreenCover(isPresented: $showOnboarding) {
    OnboardingView()
        .interactiveDismissDisabled() // 禁止手势关闭
}
```

`.interactiveDismissDisabled()` 可以禁用下滑关闭手势，强制用户完成必要操作。

### 多模态叠加

SwiftUI 支持在同一视图上叠加多个模态修饰符：

```swift
struct MultiModalView: View {
    @State private var showSheet = false
    @State private var showCover = false

    var body: some View {
        VStack {
            Button("Sheet") { showSheet = true }
            Button("全屏") { showCover = true }
        }
        .sheet(isPresented: $showSheet) { SheetView() }
        .fullScreenCover(isPresented: $showCover) { FullView() }
    }
}
```

## 26.5 编程式导航与深层链接

编程式导航是指在代码中通过逻辑控制而非用户直接点击来触发跳转。深层链接（Deep Link）则允许从外部源（如通知、URL Scheme）直接导航到应用内的特定页面。

### 基于路径的编程式导航

```swift
enum AppRoute: Hashable {
    case home
    case profile(id: UUID)
    case settings
    case detail(id: Int)
}

struct RoutedApp: View {
    @State private var path = NavigationPath()
    @State private var showSettings = false

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: AppRoute.self) { route in
                    switch route {
                    case .home:
                        HomeView()
                    case .profile(let id):
                        ProfileView(userID: id)
                    case .settings:
                        SettingsView()
                    case .detail(let id):
                        DetailView(item: id)
                    }
                }
        }
        .environment(\.showSettings, $showSettings)
        .sheet(isPresented: $showSettings) {
            SettingsView()
        }
    }
}

// 编程式跳转
func navigateToProfile(id: UUID, path: Binding<NavigationPath>) {
    path.wrappedValue.append(AppRoute.profile(id: id))
}
```

### URL Scheme 与深层链接

首先在 Info.plist 中注册自定义 URL Scheme（如 `myapp://`），然后通过 `onOpenURL` 处理：

```swift
struct DeepLinkApp: App {
    @StateObject private var router = AppRouter()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(router)
                .onOpenURL { url in
                    router.handleDeepLink(url)
                }
        }
    }
}

class AppRouter: ObservableObject {
    @Published var path = NavigationPath()
    @Published var showSheet: DeepLinkDestination?

    enum DeepLinkDestination: Identifiable {
        case profile(id: UUID)
        case post(id: Int)

        var id: String {
            switch self {
            case .profile(let id): return "profile-\(id)"
            case .post(let id): return "post-\(id)"
            }
        }
    }

    func handleDeepLink(_ url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let host = components.host else { return }

        switch host {
        case "profile":
            if let idString = components.queryItems?.first(where: { $0.name == "id" })?.value,
               let id = UUID(uuidString: idString) {
                path = NavigationPath()
                path.append(AppRoute.profile(id: id))
            }
        case "post":
            if let idString = components.queryItems?.first(where: { $0.name == "id" })?.value,
               let id = Int(idString) {
                showSheet = .post(id: id)
            }
        default:
            break
        }
    }
}
```

### Universal Links

对于生产环境，推荐使用 Universal Links（通用链接）。它需要配置 Apple App Site Association 文件，并在 Xcode 中配置 Associated Domains：

```swift
.onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
    guard let url = activity.webpageURL else { return }
    // 处理 Universal Link
    router.handleDeepLink(url)
}
```

### 推送通知中的深层链接

在推送通知中携带自定义数据，通过 `onReceive` 处理：

```swift
struct PushNotificationHandler: ViewModifier {
    @EnvironmentObject var router: AppRouter

    func body(content: Content) -> some View {
        content
            .onReceive(NotificationCenter.default.publisher(for: .didReceivePushNotification)) { notification in
                if let userInfo = notification.userInfo,
                   let routeString = userInfo["route"] as? String,
                   let route = AppRoute.from(string: routeString) {
                    router.path.append(route)
                }
            }
    }
}
```

## 本章小结

本章全面介绍了 SwiftUI 中的导航与模态展示体系。`NavigationStack` 和 `NavigationPath` 构成了值驱动的现代导航基础，`NavigationSplitView` 为 iPadOS/macOS 提供了多栏布局支持，`TabView` 是移动端顶级导航的标准选择。Sheet、Popover 和 FullScreenCover 分别对应不同程度的模态展示需求。最后，通过路径管理和 URL 处理机制，可以构建支持深层链接的完整导航解决方案。掌握这些内容，你将能够构建结构清晰、导航流畅的 SwiftUI 应用。
