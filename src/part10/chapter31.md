# 第31章 实战：构建一个 SwiftUI 应用

理论知识的最终价值在于实践。本章将带领读者从零开始构建一个完整的 SwiftUI 应用——**"TaskFlow"**，一个轻量级的任务管理工具。通过这个项目，我们将串联起前面各章所学的核心知识点，包括状态管理、视图组合、导航结构、网络层集成、动画设计以及测试策略。每一步都会给出完整的代码实现和设计思路解析。

## 31.1 设计状态模型与应用架构

### 应用功能概述

TaskFlow 的核心功能包括：
- 查看任务列表、按状态（待办/进行中/已完成）筛选
- 创建、编辑和删除任务
- 从网络获取演示任务数据
- 动画化的交互反馈

### 架构选择

TaskFlow 采用 **MVVM（Model-View-ViewModel）架构**，配合 SwiftUI 的原生状态管理系统：

- **Model**：纯数据模型，遵循 `Identifiable` 和 `Codable`
- **ViewModel**：遵循 `ObservableObject`，管理业务逻辑和状态
- **View**：SwiftUI 声明式视图，观察 ViewModel 的变化

### 模型定义

```swift
import Foundation

enum TaskStatus: String, Codable, CaseIterable {
    case todo = "待办"
    case inProgress = "进行中"
    case done = "已完成"
}

struct TaskItem: Identifiable, Codable, Equatable {
    let id: UUID
    var title: String
    var description: String
    var status: TaskStatus
    var createdAt: Date
    var dueDate: Date?
    
    init(
        id: UUID = UUID(),
        title: String,
        description: String = "",
        status: TaskStatus = .todo,
        createdAt: Date = Date(),
        dueDate: Date? = nil
    ) {
        self.id = id
        self.title = title
        self.description = description
        self.status = status
        self.createdAt = createdAt
        self.dueDate = dueDate
    }
}
```

### ViewModel 层设计

```swift
import Foundation
import Combine

@MainActor
class TaskViewModel: ObservableObject {
    @Published var tasks: [TaskItem] = []
    @Published var selectedStatus: TaskStatus? = nil
    @Published var isLoading = false
    @Published var errorMessage: String? = nil
    
    private let repository: TaskRepositoryProtocol
    
    init(repository: TaskRepositoryProtocol = TaskRepository()) {
        self.repository = repository
    }
    
    // MARK: - 计算属性
    var filteredTasks: [TaskItem] {
        guard let status = selectedStatus else {
            return tasks
        }
        return tasks.filter { $0.status == status }
    }
    
    var todoCount: Int { tasks.filter { $0.status == .todo }.count }
    var inProgressCount: Int { tasks.filter { $0.status == .inProgress }.count }
    var doneCount: Int { tasks.filter { $0.status == .done }.count }
    
    // MARK: - 数据操作
    func loadTasks() async {
        isLoading = true
        errorMessage = nil
        do {
            tasks = try await repository.fetchTasks()
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }
    
    func addTask(title: String, description: String = "") {
        let task = TaskItem(title: title, description: description)
        tasks.append(task)
    }
    
    func updateStatus(for taskId: UUID, to newStatus: TaskStatus) {
        guard let index = tasks.firstIndex(where: { $0.id == taskId }) else { return }
        tasks[index].status = newStatus
    }
    
    func deleteTask(at offsets: IndexSet) {
        tasks.remove(atOffsets: offsets)
    }
}
```

### 协议化设计

```swift
protocol TaskRepositoryProtocol {
    func fetchTasks() async throws -> [TaskItem]
    func saveTasks(_ tasks: [TaskItem]) async throws
}
```

## 31.2 组合视图与导航结构

### 导航结构设计

TaskFlow 使用 `NavigationStack` 构建导航层次：

```swift
import SwiftUI

@main
struct TaskFlowApp: App {
    @StateObject private var viewModel = TaskViewModel()
    
    var body: some Scene {
        WindowGroup {
            NavigationStack {
                TaskListView()
                    .environmentObject(viewModel)
            }
        }
    }
}
```

### 主列表视图

```swift
struct TaskListView: View {
    @EnvironmentObject var viewModel: TaskViewModel
    @State private var showAddTask = false
    
    var body: some View {
        VStack(spacing: 0) {
            statusFilterBar
            taskList
        }
        .navigationTitle("TaskFlow")
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Button(action: { showAddTask = true }) {
                    Image(systemName: "plus")
                }
            }
            ToolbarItem(placement: .cancellationAction) {
                Button("刷新") {
                    Task { await viewModel.loadTasks() }
                }
            }
        }
        .sheet(isPresented: $showAddTask) {
            AddTaskView()
        }
        .overlay {
            if viewModel.isLoading {
                ProgressView("加载中...")
            }
        }
        .alert("错误", isPresented: .constant(viewModel.errorMessage != nil)) {
            Button("确定") { viewModel.errorMessage = nil }
        } message: {
            Text(viewModel.errorMessage ?? "")
        }
        .task {
            await viewModel.loadTasks()
        }
    }
    
    // MARK: - 筛选栏
    private var statusFilterBar: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                FilterChip(title: "全部", count: viewModel.tasks.count, isSelected: viewModel.selectedStatus == nil) {
                    viewModel.selectedStatus = nil
                }
                FilterChip(title: "待办", count: viewModel.todoCount, isSelected: viewModel.selectedStatus == .todo) {
                    viewModel.selectedStatus = .todo
                }
                FilterChip(title: "进行中", count: viewModel.inProgressCount, isSelected: viewModel.selectedStatus == .inProgress) {
                    viewModel.selectedStatus = .inProgress
                }
                FilterChip(title: "已完成", count: viewModel.doneCount, isSelected: viewModel.selectedStatus == .done) {
                    viewModel.selectedStatus = .done
                }
            }
            .padding(.horizontal)
            .padding(.vertical, 8)
        }
        .background(Color(.systemGroupedBackground))
    }
    
    // MARK: - 任务列表
    private var taskList: some View {
        List {
            ForEach(viewModel.filteredTasks) { task in
                NavigationLink(destination: TaskDetailView(task: task)) {
                    TaskRowView(task: task)
                }
            }
            .onDelete { viewModel.deleteTask(at: $0) }
        }
        .listStyle(.insetGrouped)
    }
}
```

### 子视图组件

```swift
struct FilterChip: View {
    let title: String
    let count: Int
    let isSelected: Bool
    let action: () -> Void
    
    var body: some View {
        Button(action: action) {
            HStack(spacing: 4) {
                Text(title)
                    .font(.subheadline)
                Text("(\(count))")
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(isSelected ? Color.blue : Color(.systemGray6))
            .foregroundColor(isSelected ? .white : .primary)
            .clipShape(Capsule())
        }
        .buttonStyle(.plain)
    }
}

struct TaskRowView: View {
    let task: TaskItem
    
    var body: some View {
        HStack {
            statusIcon
            VStack(alignment: .leading, spacing: 4) {
                Text(task.title)
                    .font(.headline)
                    .strikethrough(task.status == .done)
                if !task.description.isEmpty {
                    Text(task.description)
                        .font(.caption)
                        .foregroundColor(.secondary)
                        .lineLimit(1)
                }
                Text(task.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundColor(.tertiary)
            }
        }
        .padding(.vertical, 4)
    }
    
    private var statusIcon: some View {
        Circle()
            .fill(statusColor)
            .frame(width: 12, height: 12)
    }
    
    private var statusColor: Color {
        switch task.status {
        case .todo: return .gray
        case .inProgress: return .blue
        case .done: return .green
        }
    }
}
```

## 31.3 集成网络层与错误处理

### Repository 实现

```swift
class TaskRepository: TaskRepositoryProtocol {
    private let baseURL = "https://api.example.com"
    private let decoder: JSONDecoder = {
        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        return decoder
    }()
    
    func fetchTasks() async throws -> [TaskItem] {
        // 模拟网络请求
        try await Task.sleep(nanoseconds: 1_000_000_000) // 1秒延迟
        
        // 模拟演示数据
        return [
            TaskItem(title: "学习 SwiftUI", description: "第29-31章", status: .inProgress),
            TaskItem(title: "完成项目报告", description: "本月工作总结", status: .todo),
            TaskItem(title: "健身打卡", description: "跑步30分钟", status: .done)
        ]
        
        // 真实网络请求示例：
        // let url = URL(string: "\(baseURL)/tasks")!
        // var request = URLRequest(url: url)
        // request.httpMethod = "GET"
        // request.setValue("application/json", forHTTPHeaderField: "Accept")
        // 
        // let (data, response) = try await URLSession.shared.data(for: request)
        // guard let httpResponse = response as? HTTPURLResponse,
        //       (200...299).contains(httpResponse.statusCode) else {
        //     throw APIError.invalidResponse
        // }
        // return try decoder.decode([TaskItem].self, from: data)
    }
    
    func saveTasks(_ tasks: [TaskItem]) async throws {
        // 持久化实现（本地 JSON 文件或云端 API）
    }
}

enum APIError: LocalizedError {
    case invalidURL
    case invalidResponse
    case decodingFailed
    
    var errorDescription: String? {
        switch self {
        case .invalidURL: return "无效的 URL"
        case .invalidResponse: return "服务器响应异常"
        case .decodingFailed: return "数据解析失败"
        }
    }
}
```

### 错误处理策略

在 TaskViewModel 中，错误被分为三类：
1. **可恢复错误**（如网络超时）：显示提示，用户可重试
2. **不可恢复错误**（如数据损坏）：显示错误视图，引导用户联系支持
3. **静默错误**（如预取数据失败）：仅在调试日志中记录

```swift
extension TaskViewModel {
    /// 带重试逻辑的数据加载
    func loadTasksWithRetry(retryCount: Int = 3) async {
        for attempt in 1...retryCount {
            isLoading = true
            do {
                tasks = try await repository.fetchTasks()
                errorMessage = nil
                isLoading = false
                return
            } catch {
                if attempt == retryCount {
                    errorMessage = "加载失败，请检查网络连接"
                    isLoading = false
                    return
                }
                // 指数退避
                try? await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(attempt))) * 500_000_000)
            }
        }
    }
}
```

## 31.4 添加动画与微交互

### 基础动画

为状态变更添加平滑过渡：

```swift
struct TaskRowView: View {
    let task: TaskItem
    
    var body: some View {
        HStack {
            statusIcon
            // ... 其他内容
        }
        .padding(.vertical, 4)
        .transition(.slide.combined(with: .opacity))
        .animation(.spring(response: 0.3, dampingFraction: 0.7), value: task.status)
    }
}
```

### 自定义过渡动画

```swift
struct AddTaskView: View {
    @Environment(\.dismiss) var dismiss
    @EnvironmentObject var viewModel: TaskViewModel
    @State private var title = ""
    @State private var description = ""
    @State private var showAnimation = false
    
    var body: some View {
        NavigationStack {
            Form {
                Section("任务信息") {
                    TextField("任务标题", text: $title)
                    TextField("任务描述", text: $description, axis: .vertical)
                        .lineLimit(3...6)
                }
                
                Section {
                    Button("创建任务") {
                        withAnimation(.spring(response: 0.4, dampingFraction: 0.6)) {
                            viewModel.addTask(title: title, description: description)
                            showAnimation = true
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
                            dismiss()
                        }
                    }
                    .disabled(title.isEmpty)
                }
            }
            .navigationTitle("新建任务")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("取消") { dismiss() }
                }
            }
            .overlay {
                if showAnimation {
                    successOverlay
                }
            }
        }
    }
    
    private var successOverlay: some View {
        Image(systemName: "checkmark.circle.fill")
            .font(.system(size: 60))
            .foregroundColor(.green)
            .transition(.scale.combined(with: .opacity))
    }
}
```

### 列表动画

```swift
// 在 TaskListView 中
ForEach(viewModel.filteredTasks) { task in
    NavigationLink(destination: TaskDetailView(task: task)) {
        TaskRowView(task: task)
    }
}
.onDelete { viewModel.deleteTask(at: $0) }
.onMove { source, destination in
    viewModel.tasks.move(fromOffsets: source, toOffset: destination)
}
.animation(.default, value: viewModel.filteredTasks)
```

### 微交互设计原则

1. **适度原则**：动画时长控制在 0.2-0.5 秒之间，过长会让用户感到延迟，过短则无法形成反馈
2. **一致性**：同类型的交互使用相同的动画曲线和时长
3. **尊重系统设置**：使用 `@Environment(\.accessibilityReduceMotion)` 检测用户是否开启了"减少动态效果"设置

```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

// 在动画中使用
withAnimation(reduceMotion ? nil : .spring()) {
    // 状态变更
}
```

## 31.5 编写可测试的视图逻辑

### 测试策略

TaskFlow 的测试分为三个层次：
1. **单元测试**：测试 ViewModel 的纯逻辑和计算属性
2. **集成测试**：测试 ViewModel 与 Repository 的交互
3. **UI 测试**：测试视图与用户的交互流程

### ViewModel 单元测试

```swift
import XCTest
@testable import TaskFlow

@MainActor
class TaskViewModelTests: XCTestCase {
    
    var viewModel: TaskViewModel!
    var mockRepository: MockTaskRepository!
    
    override func setUp() {
        super.setUp()
        mockRepository = MockTaskRepository()
        viewModel = TaskViewModel(repository: mockRepository)
    }
    
    override func tearDown() {
        viewModel = nil
        mockRepository = nil
        super.tearDown()
    }
    
    func testAddTask() {
        // Given
        XCTAssertEqual(viewModel.tasks.count, 0)
        
        // When
        viewModel.addTask(title: "测试任务", description: "测试描述")
        
        // Then
        XCTAssertEqual(viewModel.tasks.count, 1)
        XCTAssertEqual(viewModel.tasks.first?.title, "测试任务")
        XCTAssertEqual(viewModel.tasks.first?.description, "测试描述")
        XCTAssertEqual(viewModel.tasks.first?.status, .todo)
    }
    
    func testUpdateTaskStatus() {
        // Given
        viewModel.addTask(title: "任务1")
        let taskId = viewModel.tasks[0].id
        
        // When
        viewModel.updateStatus(for: taskId, to: .done)
        
        // Then
        XCTAssertEqual(viewModel.tasks.first?.status, .done)
    }
    
    func testDeleteTask() {
        // Given
        viewModel.addTask(title: "任务1")
        viewModel.addTask(title: "任务2")
        
        // When
        viewModel.deleteTask(at: IndexSet(integer: 0))
        
        // Then
        XCTAssertEqual(viewModel.tasks.count, 1)
        XCTAssertEqual(viewModel.tasks.first?.title, "任务2")
    }
    
    func testFilteredTasks() {
        // Given
        viewModel.addTask(title: "待办任务")
        viewModel.updateStatus(for: viewModel.tasks[0].id, to: .todo)
        viewModel.addTask(title: "进行中任务")
        viewModel.updateStatus(for: viewModel.tasks[1].id, to: .inProgress)
        
        // When
        viewModel.selectedStatus = .inProgress
        
        // Then
        XCTAssertEqual(viewModel.filteredTasks.count, 1)
        XCTAssertEqual(viewModel.filteredTasks.first?.title, "进行中任务")
    }
    
    func testLoadTasks() async {
        // Given
        mockRepository.mockTasks = [
            TaskItem(title: "Mock 任务1"),
            TaskItem(title: "Mock 任务2")
        ]
        
        // When
        await viewModel.loadTasks()
        
        // Then
        XCTAssertEqual(viewModel.tasks.count, 2)
        XCTAssertFalse(viewModel.isLoading)
        XCTAssertNil(viewModel.errorMessage)
    }
    
    func testLoadTasksFailure() async {
        // Given
        mockRepository.shouldThrow = true
        
        // When
        await viewModel.loadTasks()
        
        // Then
        XCTAssertTrue(viewModel.tasks.isEmpty)
        XCTAssertFalse(viewModel.isLoading)
        XCTAssertNotNil(viewModel.errorMessage)
    }
}

// MARK: - Mock Repository
class MockTaskRepository: TaskRepositoryProtocol {
    var shouldThrow = false
    var mockTasks: [TaskItem] = []
    
    func fetchTasks() async throws -> [TaskItem] {
        if shouldThrow {
            throw APIError.invalidResponse
        }
        return mockTasks
    }
    
    func saveTasks(_ tasks: [TaskItem]) async throws {
        mockTasks = tasks
    }
}
```

### 可测试性设计原则

1. **依赖注入**：ViewModel 通过协议接收 Repository，测试时注入 MockRepository
2. **纯函数优先**：尽量将业务逻辑提取为不依赖 SwiftUI 上下文的纯函数
3. **异步隔离**：使用 `@MainActor` 确保 ViewModel 的状态更新在主线程进行
4. **边界测试**：特别关注空列表、重复数据、超大标题等边界情况

### 完整的应用架构图

```
┌─────────────────────────────────────────────────┐
│                     View 层                       │
│  TaskListView → TaskRowView → TaskDetailView    │
│  AddTaskView → FilterChip                      │
│  观察 @EnvironmentObject / @ObservedObject       │
└──────────────────────┬──────────────────────────┘
                       │ 观察
┌──────────────────────▼──────────────────────────┐
│                 ViewModel 层                     │
│  TaskViewModel (ObservableObject)              │
│  @Published tasks, selectedStatus, isLoading   │
│  filteredTasks, loadTasks(), addTask()...       │
└──────────────────────┬──────────────────────────┘
                       │ 调用
┌──────────────────────▼──────────────────────────┐
│                Repository 层                     │
│  TaskRepository : TaskRepositoryProtocol        │
│  fetchTasks() → 网络请求或本地缓存               │
│  saveTasks() → 持久化存储                        │
└─────────────────────────────────────────────────┘
```

## 小结

本章通过构建 TaskFlow 这个完整的任务管理应用，将 SwiftUI 的核心知识串联成了完整的工程实践。我们从架构设计开始，定义了清晰的数据模型和协议化接口；然后通过组合视图和 `NavigationStack` 搭建了直观的导航结构；接着集成了网络层并设计了健壮的错误处理机制；再通过动画和微交互提升了用户体验；最后，编写了全面的单元测试来确保业务逻辑的正确性。

这个实战项目虽然精简，但涵盖了真实应用开发中的大多数关键环节。建议读者在自己的项目中尝试重构和扩展 TaskFlow——例如添加 Core Data 本地持久化、Push Notification 推送通知、或者使用 WidgetKit 添加桌面小组件。实践是掌握 SwiftUI 的最佳途径，希望本章能成为你从学习者到实践者的桥梁。
