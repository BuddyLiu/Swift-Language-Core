# 第29章 与 UIKit/AppKit 互操作

SwiftUI 作为 Apple 于 2019 年推出的新一代声明式 UI 框架，虽然已经覆盖了大多数日常开发场景，但在实际项目中，我们仍然不可避免地需要与 UIKit（iOS/macOS Catalyst）或 AppKit（macOS）进行互操作。这种需求可能源于以下原因：需要使用尚未被 SwiftUI 封装的原生控件、复用已有的庞大代码库、或者调用某些只有 UIKit/AppKit 才提供的底层 API。本章将深入探讨 SwiftUI 与 UIKit/AppKit 之间的互操作机制，包括 `UIViewRepresentable`/`NSViewRepresentable` 协议的使用、视图控制器的包装、迁移策略以及混编项目中的状态同步方案。

## 29.1 UIViewRepresentable 与 NSViewRepresentable

### 协议本质

`UIViewRepresentable` 和 `NSViewRepresentable` 是 SwiftUI 为开发者提供的"逃逸口"。它们的核心思想是：**将一个 UIKit/AppKit 的视图（View）包装成一个 SwiftUI 的视图（View）**，从而让这个传统视图可以无缝地嵌入到 SwiftUI 的视图层次结构中。

这两个协议的签名几乎相同，以 `UIViewRepresentable` 为例：

```swift
public protocol UIViewRepresentable : View where Self.Body == Never {
    /// 创建并返回一个 UIView 实例。
    associatedtype UIViewType : UIView
    func makeUIView(context: Self.Context) -> Self.UIViewType
    
    /// 更新视图的状态，当 SwiftUI 的状态发生变化时调用。
    func updateUIView(_ uiView: Self.UIViewType, context: Self.Context)
    
    /// 创建一个协调器对象，用于处理 UIKit 的代理/目标-动作回调。
    associatedtype Coordinator = Void
    func makeCoordinator() -> Self.Coordinator
}
```

### 核心方法

- **`makeUIView(context:)`**：负责创建底层的 UIKit 控件实例。这个方法仅在视图首次被放入视图树时调用一次。
- **`updateUIView(_:context:)`**：当 SwiftUI 的 `@State`、`@Binding` 或外部数据发生变化时，SwiftUI 会调用此方法，开发者在此方法中将最新的数据同步到 UIKit 控件上。
- **`makeCoordinator()`**：创建一个"协调器"对象，充当 UIKit 控件的事件委托（Delegate）或 Target-Action 的目标。协调器通常是一个嵌套类，持有对父 `Representable` 的引用，从而将 UIKit 的回调转化为 SwiftUI 可以理解的数据流。

### 实际示例：包装 UITextField

```swift
struct SwiftUITextField: UIViewRepresentable {
    @Binding var text: String
    var placeholder: String = ""
    
    func makeUIView(context: Context) -> UITextField {
        let textField = UITextField(frame: .zero)
        textField.placeholder = placeholder
        textField.delegate = context.coordinator
        textField.borderStyle = .roundedRect
        textField.addTarget(
            context.coordinator,
            action: #selector(Coordinator.textFieldDidChange(_:)),
            for: .editingChanged
        )
        return textField
    }
    
    func updateUIView(_ uiView: UITextField, context: Context) {
        // 只有当 textField 的文字与当前绑定值不同时才更新，避免光标跳动
        if uiView.text != text {
            uiView.text = text
        }
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text)
    }
    
    class Coordinator: NSObject, UITextFieldDelegate {
        @Binding var text: String
        
        init(text: Binding<String>) {
            self._text = text
        }
        
        @objc func textFieldDidChange(_ sender: UITextField) {
            text = sender.text ?? ""
        }
        
        func textFieldShouldReturn(_ textField: UITextField) -> Bool {
            textField.resignFirstResponder()
            return true
        }
    }
}
```

### NSViewRepresentable 示例

对于 macOS 平台，`NSViewRepresentable` 的使用模式完全一致：

```swift
struct SwiftUIWebView: NSViewRepresentable {
    let url: URL
    
    func makeNSView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        return webView
    }
    
    func updateNSView(_ nsView: WKWebView, context: Context) {
        let request = URLRequest(url: url)
        nsView.load(request)
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator()
    }
    
    class Coordinator: NSObject, WKNavigationDelegate {
        // 处理网页加载事件
    }
}
```

### 生命周期管理

在 UIKit 中，视图的生命周期由视图控制器管理，而 `UIViewRepresentable` 的生命周期则完全由 SwiftUI 接管。当 SwiftUI 视图被销毁时，对应的 UIKit 视图及其协调器也会被清理。开发者无需手动管理释放操作。但需要注意：如果协调器中持有对外部对象的强引用，可能会导致循环引用，应当使用弱引用加以规避。

## 29.2 包装 UIViewController

某些 UIKit 组件以视图控制器（UIViewController）的形式存在，例如 `UIImagePickerController`、`UIDocumentPickerViewController` 或 `MFMailComposeViewController`。SwiftUI 提供了 `UIViewControllerRepresentable` 协议来包装这些控制器。

### UIViewControllerRepresentable 协议

```swift
public protocol UIViewControllerRepresentable : View where Self.Body == Never {
    associatedtype UIViewControllerType : UIViewController
    func makeUIViewController(context: Self.Context) -> Self.UIViewControllerType
    func updateUIViewController(_ uiViewController: Self.UIViewControllerType, context: Self.Context)
    associatedtype Coordinator = Void
    func makeCoordinator() -> Self.Coordinator
}
```

### 包装 UIImagePickerController

```swift
struct ImagePicker: UIViewControllerRepresentable {
    @Binding var selectedImage: UIImage?
    @Environment(\.dismiss) var dismiss
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .photoLibrary
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {
        // 无需更新
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: ImagePicker
        
        init(_ parent: ImagePicker) {
            self.parent = parent
        }
        
        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.selectedImage = image
            }
            parent.dismiss()
        }
        
        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}
```

### 使用方式

```swift
struct ContentView: View {
    @State private var showPicker = false
    @State private var image: UIImage?
    
    var body: some View {
        VStack {
            if let image = image {
                Image(uiImage: image)
                    .resizable()
                    .scaledToFit()
            }
            Button("选择图片") {
                showPicker = true
            }
        }
        .sheet(isPresented: $showPicker) {
            ImagePicker(selectedImage: $image)
        }
    }
}
```

## 29.3 从 UIKit 到 SwiftUI 的迁移策略

从 UIKit 迁移到 SwiftUI 是一个渐进式的过程，而非"大爆炸"式的重写。以下是经过验证的几种迁移路径。

### 策略一：自下而上（Leaf-First）

从最底层的 UI 组件（如单个按钮、标签、列表项）开始，将它们用 `UIViewRepresentable` 或纯 SwiftUI 重写，然后逐步向上替换。这种方式的优点是每次改动范围小、风险低，适合团队逐步学习 SwiftUI。

### 策略二：自上而下（Screen-First）

先使用 SwiftUI 搭建整个页面的框架（NavigationView、TabView），然后在需要复杂交互或尚未支持的功能时，使用 `UIViewControllerRepresentable` 嵌入原有的 UIViewController。这种方式让团队能快速体验 SwiftUI 的开发模式，同时保留旧代码。

### 策略三：模块化迁移

按照功能模块进行划分，将某些完整的 Feature（如设置页面、用户信息编辑页）整体迁移至 SwiftUI。可以使用 `UIHostingController` 将 SwiftUI 视图嵌入到 UIKit 中：

```swift
// 在 UIKit 中嵌入 SwiftUI 视图
let swiftUIView = SettingsView()
let hostingController = UIHostingController(rootView: swiftUIView)
navigationController.pushViewController(hostingController, animated: true)
```

### 混合项目中需要注意的问题

1. **导航栈一致性**：避免在 UIKit 的 push 和 SwiftUI 的 NavigationLink 之间频繁切换，容易导致导航栈混乱。
2. **生命周期差异**：UIKit 的 `viewWillAppear` / `viewDidDisappear` 与 SwiftUI 的 `onAppear` / `onDisappear` 在触发时机上存在细微差别，迁移时需注意。
3. **响应链**：UIResponder 链在混合项目中可能被打破，某些系统级行为（如键盘、菜单）可能表现异常。

## 29.4 混编项目中的状态同步

状态同步是混编项目中最容易出问题的环节。SwiftUI 的数据驱动模式与 UIKit 的事件驱动模式存在根本差异，必须建立清晰的桥梁。

### 使用 ObservableObject 桥接

```swift
class AppState: ObservableObject {
    @Published var isLoggedIn = false
    @Published var userName = ""
}

// 在 UIKit 中订阅
class UserProfileViewController: UIViewController {
    private var cancellables = Set<AnyCancellable>()
    let appState = AppState.shared
    
    override func viewDidLoad() {
        super.viewDidLoad()
        appState.$userName
            .sink { [weak self] name in
                self?.userNameLabel.text = name
            }
            .store(in: &cancellables)
    }
}
```

### 使用 NotificationCenter 桥接

对于某些一次性或松耦合的事件传递，可以使用 `NotificationCenter`：

```swift
// SwiftUI 侧发送通知
Button("操作完成") {
    NotificationCenter.default.post(
        name: .init("com.app.taskCompleted"),
        object: nil,
        userInfo: ["taskId": task.id]
    )
}

// UIKit 侧接收
NotificationCenter.default.addObserver(
    self,
    selector: #selector(handleTaskCompleted(_:)),
    name: .init("com.app.taskCompleted"),
    object: nil
)
```

### 共享数据层

最佳实践是将业务逻辑和数据层完全独立于 UI 框架。无论是 SwiftUI 还是 UIKit，都通过统一的 Repository / Service 层获取数据。这样，状态同步问题就变成了"同一个数据源的双向绑定"问题：

```swift
// 统一的数据层
class TaskRepository: ObservableObject {
    @Published var tasks: [Task] = []
    
    func fetchTasks() async throws {
        // 网络请求
    }
    
    func updateTask(_ task: Task) async throws {
        // 更新任务
    }
}

// SwiftUI 使用
struct TaskListView: View {
    @StateObject private var repository = TaskRepository()
    var body: some View { /* ... */ }
}

// UIKit 使用
class TaskListViewController: UIViewController {
    let repository = TaskRepository()
    private var cancellables = Set<AnyCancellable>()
    
    override func viewDidLoad() {
        repository.$tasks
            .receive(on: DispatchQueue.main)
            .sink { [weak self] tasks in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
    }
}
```

### 互操作注意事项清单

| 场景 | 推荐方案 | 避免的做法 |
|------|----------|-----------|
| 包装原生控件 | UIViewRepresentable | 直接修改 UIKit 视图的 frame |
| 状态传递 | @Binding / ObservableObject | 使用全局变量 |
| 事件回调 | Coordinator / Delegate | 在 Representable 中直接设置 target-action |
| 生命周期 | onAppear / onDisappear | 依赖 viewDidAppear 时序 |
| 导航 | NavigationStack + UIHostingController | 混用两种 push 方式 |

## 小结

SwiftUI 与 UIKit/AppKit 的互操作是实际项目中不可或缺的能力。本章详细介绍了 `UIViewRepresentable` / `NSViewRepresentable` 以及 `UIViewControllerRepresentable` 的使用方法，分析了三种主流的迁移策略，并提供了混编项目中的状态同步方案。掌握这些技术，开发者可以在享受 SwiftUI 声明式开发效率的同时，无缝复用已有的 UIKit/AppKit 代码资产，实现平滑的技术栈演进。

在下一章中，我们将进一步探讨 SwiftUI 的性能优化与最佳实践，帮助你在实际项目中写出既优雅又高效的应用。
