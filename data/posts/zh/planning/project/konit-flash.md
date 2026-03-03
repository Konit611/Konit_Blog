---
title: KonitFlash - 用 SwiftUI + SwiftData 打造 Anki 风格间隔重复应用
date: 2026-03-03 19:45
excerpt: SM-2 算法、iCloud 同步、VIP 架构、CSV 导入 — iOS/macOS 闪卡应用背后的技术选型与实现过程。
coverImage:
categories:
  - project
tags:
  - SwiftUI
  - SwiftData
  - CloudKit
  - iOS
  - VIP
author: Geunil Park
featured: false
---

## 为什么要做

用 NotebookLM 做学习资料，再导入 Anki，每次都要花不少时间。导出 CSV、调格式、导入。花在准备上的时间比实际学习还多。

Anki 本身也有不满。功能强大，但移动端体验停留在 2010 年代。想在 iPhone 上做的卡片无缝切到 Mac 继续复习，设置起来很麻烦。

我想要的是这样的：

- 从 NotebookLM CSV **一键导入**
- 基于 SM-2 算法的**间隔重复**能正常运作
- 通过 iCloud 实现 iPhone ↔ Mac **自动同步**
- 深色主题的**简洁 UI**

市面上没有完全满足的，于是决定自己做。

<!-- TODO: 应用截图 3 张（首页、学习、结果） -->
<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ko/planning/project/konit-flash-home.jpg" alt="首页" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-study.jpg" alt="学习界面" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-result.jpg" alt="结果界面" style="width: 32%; border-radius: 8px;" />
</div>

---

## 技术栈一览

| 领域 | 技术 |
|------|------|
| UI | SwiftUI、`horizontalSizeClass` 响应式 |
| 数据 | SwiftData（`@Model`） |
| 架构 | VIP（View-Interactor-Presenter） |
| 算法 | SM-2 Spaced Repetition |
| 同步 | CloudKit（SwiftData 自动同步） |
| 多语言 | String Catalog（en/ko/ja/zh-Hans） |
| 小组件 | WidgetKit + App Groups |
| 测试 | Swift Testing（XCTest 兼容） |

基于 SwiftUI + SwiftData，目标平台 iOS 18.6+ / macOS 15.0+。

---

## 架构：VIP 模式

### 为什么选 VIP

MVVM 和 SwiftUI 天然契合，但 ViewModel 容易膨胀。业务逻辑和视图映射逻辑混在一起，测试变难，代码也不好读。

VIP 通过**三层分离**来解决这个问题：

```
View ──── Presenter ──── Interactor
(UI)      (映射)         (业务逻辑)
```

- **View**：纯 UI。只绑定 Presenter 的 `@Published` 属性。
- **Presenter**：`ObservableObject`。将领域数据转换为 ViewState。
- **Interactor**：纯业务逻辑。通过 DI 接收 `ModelContext`，执行 CRUD，返回 **plain struct**。

### 实际代码中的 VIP

**Interactor** — 接收 ModelContext，查询数据并返回 plain struct：

```swift
final class HomeInteractor {
    private let modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    func fetchHomeData() -> HomeData {
        let decks = fetchDecks()
        let recentLogs = fetchRecentLogs()
        let stats = computeStats(decks: decks, recentLogs: recentLogs)
        let weeklyActivities = computeWeeklyActivity(allLogs: recentLogs)
        return HomeData(stats: stats, weeklyActivities: weeklyActivities, decks: decks)
    }
}
```

**Presenter** — 将 Interactor 的输出映射为 ViewState：

```swift
final class HomePresenter: ObservableObject {
    @Published var viewState = HomeViewState()
    private var interactor: HomeInteractor?

    func configure(modelContext: ModelContext) {
        if interactor != nil { loadData(); return }
        self.interactor = HomeInteractor(modelContext: modelContext)
        loadData()
    }

    func loadData() {
        guard let interactor else { return }
        let data = interactor.fetchHomeData()
        viewState = HomeViewState(
            stats: mapStats(data.stats),
            weeklyData: mapWeeklyData(data.weeklyActivities),
            decks: mapDecks(data.decks),
            isEmpty: data.decks.isEmpty
        )
    }
}
```

**View** — 以 `@StateObject` 持有 Presenter，只负责 UI：

```swift
struct HomeView: View {
    @StateObject private var presenter = HomePresenter()
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        ScrollView {
            StatsSectionView(stats: presenter.viewState.stats)
            WeeklyChartView(data: presenter.viewState.weeklyData)
            // ...
        }
        .onAppear {
            presenter.configure(modelContext: modelContext)
        }
    }
}
```

### Deferred Configure 模式

SwiftUI 中无法在 View 的 `init` 里访问 `@Environment(\.modelContext)`。因此采用无参创建 Presenter，在 `.onAppear` 中调用 `configure(modelContext:)` 的**延迟初始化**模式。

```swift
// ❌ init 中无法访问 @Environment
init() {
    let interactor = HomeInteractor(modelContext: modelContext) // 编译错误
}

// ✅ 在 onAppear 中注入
.onAppear {
    presenter.configure(modelContext: modelContext)
}
```

重入时（`configure` 被调用两次的情况）复用已有的 Interactor，只重新执行 `loadData()`。

### 不用协议，保持简洁

只有一个实现的情况下，协议是不必要的抽象。KonitFlash 选择了**不使用协议**直接引用的方式。测试中注入 in-memory `ModelContainer` 创建隔离环境就够了。

---

## 数据层：SwiftData

### 模型设计

三个 `@Model` 类是核心：

```swift
@Model final class Deck {
    var id: UUID = UUID()
    var name: String = ""
    var deckDescription: String = ""
    var colorTag: String = "pink"
    @Relationship(deleteRule: .cascade, inverse: \Card.deck)
    var cards: [Card]?
}

@Model final class Card {
    var id: UUID = UUID()
    var front: String = ""
    var back: String = ""
    var deck: Deck?

    // SM-2 状态字段
    var dueDate: Date = Date()
    var interval: Double = 0         // 到下次复习的天数
    var easeFactor: Double = 2.5     // 难度系数
    var repetitions: Int = 0         // 学习重复次数
    var box: Int = 1                 // Leitner 盒子（1~5）
}

@Model final class StudyLog {
    var id: UUID = UUID()
    var card: Card?
    var grade: Int = 0               // 0=again, 1=hard, 2=good, 3=easy
    var studiedAt: Date = Date()
    var elapsedSeconds: Double = 0   // 每张卡片耗时
}
```

**设计要点：**

- **级联删除**：删除 Deck 时自动删除其下的 Card，删除 Card 时 StudyLog 也一并删除
- **所有字段都有默认值**：CloudKit 兼容所必需。有默认值才能在同步冲突时保持安全
- **`deckDescription`**：`description` 与 Swift 的 `CustomStringConvertible` 冲突，所以回避

### CloudKit 自动同步

SwiftData 的 CloudKit 集成一行搞定：

```swift
let config = ModelConfiguration(
    schema: schema,
    cloudKitDatabase: .automatic  // 就这一行
)
let container = try ModelContainer(for: schema, configurations: [config])
```

但实战中有几个坑：

1. **不支持 `unique` 约束** — CloudKit 不支持 unique constraint。必须在应用层处理重复。不知道这点就在模型里加了 `#Unique`，CloudKit 一激活就崩溃，折腾了好一阵。
2. **冲突解决是 Last-Writer-Wins** — 没有其他策略可选。必须接受 iPhone 上修改的卡片可能被 Mac 覆盖的情况。
3. **离线优先** — 在本地 SwiftData 上正常运行，网络恢复时自动同步。这是优点。

这些限制必须在模型设计前搞清楚，否则后面就是迁移地狱。应用启动时按 CloudKit → 本地 → 内存的顺序构建了**降级链**，确保任何环境下都能运行：

```swift
init() {
    do {
        // 第一优先：CloudKit
        container = try ModelContainer(for: schema,
            configurations: [ModelConfiguration(cloudKitDatabase: .automatic)])
    } catch {
        do {
            // 第二：本地
            container = try ModelContainer(for: schema,
                configurations: [ModelConfiguration(cloudKitDatabase: .none)])
        } catch {
            // 最后手段：内存
            container = try! ModelContainer(for: schema,
                configurations: [ModelConfiguration(isStoredInMemoryOnly: true)])
        }
    }
}
```

---

## SM-2 间隔重复算法

### 什么是 SM-2

SM-2（SuperMemo 2）是 1987 年 Piotr Wozniak 开发的间隔重复算法。核心思路很简单：

- **容易的卡片** → 增加复习间隔
- **困难的卡片** → 缩短复习间隔
- **Ease Factor（EF）** — 每张卡片的难度系数。随着重复不断调整

### 实现

`SRSEngine` 是用纯函数实现的 `enum`。没有副作用，测试很方便。

```swift
enum SRSEngine {
    static let minimumEF: Double = 1.3   // EF 下限
    static let maximumInterval: Double = 365  // 最长 1 年

    static func compute(
        grade: AnswerGrade,
        currentInterval: Double,
        currentEF: Double,
        currentRepetitions: Int,
        currentBox: Int
    ) -> SRSResult {
        switch grade {
        case .again:
            // 完全重置：1 分钟后重试，repetitions = 0
            newInterval = 1.0 / (24.0 * 60.0)
            newRepetitions = 0

        case .hard:
            // 间隔增加 20%，EF 减少
            newInterval = max(1, currentInterval * 1.2)
            newEF = max(minimumEF, currentEF - 0.15)

        case .good:
            // 标准 SM-2：interval × EF
            if currentRepetitions == 0 { newInterval = 1 }      // 1 天
            else if currentRepetitions == 1 { newInterval = 6 }  // 6 天
            else { newInterval = currentInterval * currentEF }

        case .easy:
            // SM-2 + 30% 加成，EF 增加
            newInterval = currentInterval * currentEF * 1.3
            newEF = min(3.0, currentEF + 0.15)
        }
    }
}
```

### 学习会话逻辑

不是简单地展示卡片打分，而是管理一个**学习队列（Learning Queue）**：

```
[Review Queue] ─── 正常复习对象卡片
[Learning Queue] ── Again/Hard 后等待重学的卡片（基于 readyAt 时间）
```

1. Learning Queue 中有 `readyAt <= now` 的卡片时优先出示
2. 没有则从 Review Queue 取下一张
3. 两个队列都空但 Learning Queue 有等待中的卡片，返回**等待状态**
4. 全部完成后跳转到结果页面

按下 "Again" 后卡片会在 1 分钟后再次出现，在同一会话内可以反复练习错误的卡片。这个流程受到了 Anki 的学习步骤模型启发。

### 预览间隔

回答按钮上显示预期的复习间隔，方便用户判断：

```
[Again: <1分] [Hard: 1天] [Good: 6天] [Easy: 12天]
```

`SRSEngine.previewIntervals()` 计算 4 个等级各自的预期间隔并以字符串返回。

<!-- TODO: 学习界面按钮区域截图 -->

---

## CSV 导入：NotebookLM 联动

从 Google NotebookLM 导出学习资料为 CSV/TSV，就能直接导入 KonitFlash。

### CSVParser 实现

RFC 4180 兼容 + 处理实战中遇到的边缘情况：

```swift
enum CSVParser {
    static func parse(_ content: String) -> CSVParseResult {
        // 1. 去除 UTF-8 BOM（Excel 常见）
        // 2. 自动检测分隔符（制表符 vs 逗号）
        // 3. 自动跳过标题行（"Term"、"Definition" 等）
        // 4. RFC 4180 引号转义处理
        // 5. 跳过空行 + 列数不匹配警告
    }
}
```

**自动分隔符检测**：比较第一行中制表符和逗号的出现频率。NotebookLM 主要用 TSV，Excel 用 CSV，两种都要支持。

**标题检测**：检测 "Term"、"Definition"、"Front"、"Back"、"Question"、"Answer" 等常见标题关键词，自动跳过第一行。

### 导入流程

```
选择文件（.fileImporter）
    ↓
CSVParser.parse() → 预览（前 5 张 + 共 N 张）
    ↓
确认 → Interactor.importCards() → ModelContext 保存
```

预览阶段展示解析错误和跳过的行数，让用户了解问题并决定是否继续。

<!-- TODO: CSV 导入预览界面截图 -->

---

## 响应式 UI：一套代码适配 iPhone 和 Mac

### 用 `horizontalSizeClass` 分流

利用 SwiftUI 的 `horizontalSizeClass`，让所有界面适配 iPhone 和 Mac：

```swift
@Environment(\.horizontalSizeClass) private var sizeClass
private var isRegular: Bool { sizeClass == .regular }

// 字体大小
.font(.system(size: isRegular ? 64 : 36, weight: .bold))

// 布局
if isRegular {
    LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: 3)) {
        ForEach(decks) { deck in DeckCardView(deck: deck) }
    }
} else {
    ForEach(decks) { deck in DeckCardView(deck: deck) }
}

// 内边距
.padding(.horizontal, isRegular ? 40 : 15)
```

在 `ContentView` 中用 `GeometryReader` 宽度 1000pt 为界手动设置 `compact`/`regular`。不用系统默认值而手动控制，是为了精确掌控 iPad 上的表现。

<!-- TODO: iPhone / Mac 对比截图并排 -->

### Mac 专属考量

Mac 上添加了键盘快捷键：

- **Space**：翻转卡片
- **1~4**：Again / Hard / Good / Easy 评级选择

用 `#if os(macOS)` 分流，隔离平台专属代码。

---

## 设计系统

### 颜色令牌

在深色背景（`#040422`）上使用强调色的设计：

| 令牌 | 颜色 | 用途 |
|------|------|------|
| `appBackground` | `#040422` | 全局背景 |
| `overdueText` | `#EF3E3E` | Overdue 横幅、Again 按钮 |
| `streakPink` | `#FFC7EA` | Streak、Hard 按钮、进度条 |
| `learnedGreen` | `#D4F849` | Learned、Good 按钮、Start 按钮 |
| `weeklyMint` | `#9CF2E8` | Weekly 图表、Easy 按钮 |

用 Color Extension 支持 hex 初始化，`ColorTag` enum 映射每个卡组的颜色。

<!-- TODO: 设计系统颜色令牌实际界面截图 -->

### 复用组件

15 个共享组件整理在 `DesignSystem/Components/` 中。介绍几个关键模式。

**3D 卡片翻转：**

```swift
.rotation3DEffect(.degrees(isFlipped ? 180 : 0), axis: (x: 0, y: 1, z: 0))
.animation(.easeInOut(duration: 0.4), value: isFlipped)
```

正面和背面分别用 `opacity` 切换，翻转过程中两面自然过渡。

**防止布局跳动：**

用 `if` 条件添加/移除元素会导致布局跳动。改为始终渲染，用 `opacity` 和 `allowsHitTesting` 控制：

```swift
AnswerButtonRow(intervals: intervals, onSelect: onGrade)
    .opacity(isFlipped ? 1 : 0)
    .allowsHitTesting(isFlipped)
```

**按压反馈：**

```swift
struct CardPressStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .scaleEffect(configuration.isPressed ? 0.97 : 1)
            .opacity(configuration.isPressed ? 0.9 : 1)
    }
}
```

---

## 多语言支持

### 运行时语言切换

可以在应用内切换语言，不受系统语言影响。`LanguageManager` 通过 `@Published` 发布当前 `Bundle`，Presenter 订阅它，在语言变更时自动刷新 ViewState。

```swift
// Presenter 中订阅语言变更
LanguageManager.shared.$locale
    .dropFirst()
    .receive(on: RunLoop.main)
    .sink { [weak self] _ in self?.loadData() }
    .store(in: &cancellables)
```

使用 String Catalog（`.xcstrings`）支持 en、ko、ja、zh-Hans 4 种语言。

---

## 测试策略

受 Vladimir Khorikov 的 **Unit Testing Principles, Practices, and Patterns** 启发的测试策略。

### 核心原则

- 测试**可观察的行为**，不测试内部实现
- **黑盒测试**：只验证输入 → 输出
- **Mock 仅用于共享依赖**：用 in-memory `ModelContainer` 隔离数据库

### 测试优先级

1. **SRSEngine** — 最关键的业务逻辑。验证所有评级组合的 interval/EF 计算
2. **CSVParser** — 正常/异常 CSV、BOM、引号转义
3. **Interactor** — CRUD 操作（in-memory ModelContainer）
4. **Presenter** — ViewState 映射验证

```swift
// SRSEngine 测试示例
@Test func againResetsRepetitions() {
    let result = SRSEngine.compute(
        grade: .again,
        currentInterval: 10,
        currentEF: 2.5,
        currentRepetitions: 3,
        currentBox: 4
    )
    #expect(result.repetitions == 0)
    #expect(result.box == 1)
    #expect(result.interval < 0.001)  // 约 1 分钟
}
```

SwiftData 测试使用 in-memory `ModelContainer` 保证测试间隔离：

```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)
let container = try ModelContainer(for: Deck.self, Card.self, StudyLog.self,
                                    configurations: config)
let context = ModelContext(container)
let interactor = HomeInteractor(modelContext: context)
```

---

## 实战中总结的 Tips

### `navigationDestination(isPresented:)` 只能用一个

在一个 View 中使用多个 `navigationDestination(isPresented:)` 会触发 iOS 导航 bug。解决方法是用本地 enum 分流：

```swift
enum DetailNavTarget {
    case flashCard(UUID)
    case editCard(UUID, UUID)
}

@State private var navTarget: DetailNavTarget?
@State private var showNav = false

// 用一个 navigationDestination 处理多个目的地
.navigationDestination(isPresented: $showNav) {
    switch navTarget {
    case .flashCard(let id): FlashCardView(deckID: id)
    case .editCard(let deckID, let cardID): AddCardView(deckID: deckID, editingCardID: cardID)
    case nil: EmptyView()
    }
}
```

### 编辑模式复用

"添加" 和 "编辑" 的 UI 几乎一样。不另建页面，复用同一个 View：

```swift
struct AddDeckView: View {
    let editingDeckID: UUID?  // nil = 创建，UUID = 编辑

    init(editingDeckID: UUID? = nil) {
        self.editingDeckID = editingDeckID
    }
}
```

Presenter 中判断 `isEditMode`，动态切换标题和按钮文本。

### Debounced Search with Combine

每次搜索词变化都执行查询会有性能问题。用 Combine 的 `debounce` 加 200ms 延迟：

```swift
$searchText
    .dropFirst()
    .debounce(for: .milliseconds(200), scheduler: RunLoop.main)
    .sink { [weak self] _ in self?.applySearch() }
    .store(in: &cancellables)
```

### 小组件数据同步

WidgetKit 在独立进程中运行。通过 App Groups 的 `UserDefaults` 共享数据，应用每次回到前台时刷新小组件时间线。

---

## 复盘 — 说实话

### 用数字看项目

| 项目 | 数值 |
|------|------|
| Swift 文件 | 62 个 |
| 代码行数 | 约 8,000 行 |
| 页面（Scene） | 8 个 |
| 复用组件 | 15 个 |
| 支持语言 | 4 种（en、ko、ja、zh-Hans） |
| 测试套件 | 6 个 |
| 开发周期 | Phase 0~5（UI 原型 → 小组件/测试） |

### 做得好的

**不用协议轻量化应用 VIP。** Bullet Journal 中用了基于协议的 DI，但对于个人项目来说是额外负担。这次直接引用实现 + in-memory ModelContainer 测试，保持简洁，测试也没问题。

**把 SM-2 隔离为纯函数 enum。** 算法是整个应用最关键的业务逻辑，做成无副作用的结构，只需测试输入 → 输出，修改时很安心。

**CSV 导入的用户体验。** 加上自动分隔符检测、标题跳过和预览后，只需 "选择文件 → 确认" 两步就完成了。正是最初想要实现的那个流程。

### 遗憾的

**太晚发现 CloudKit 的限制。** 模型设计完才知道 `unique` 约束不能用。如果早期用 PoC 先测试 CloudKit 同步，模型结构可能会不同。又一次验证了使用新框架时要先验证核心功能这个教训。

**测试覆盖偏向 SRSEngine 和 CSVParser。** Interactor 和 Presenter 的测试从结构上可以写，但赶着实现功能就一直推迟。"搭好了可测试的架构却不写测试" 这个讽刺和 Bullet Journal 时一模一样。

**多语言字符串审校。** 支持 4 种语言，但日语和中文没有经过母语者审校。功能上能用，但表达是否自然需要确认。

---

## 最后

做这个应用最大的收获是明白了**"约束决定设计"**这件事。

CloudKit 不支持 `unique` 改变了模型设计，SwiftUI 的 `@Environment` 访问限制催生了 Deferred Configure 模式，iOS 的 `navigationDestination` bug 迫使采用 enum 路由。最初预想的设计一个都没留下来。全是撞上约束后产生的结果。

不是完美的应用。测试还要多写，多语言审校也还没做完。但 "从 NotebookLM 导出 CSV，一键导入，用 SM-2 复习" 这个最初想要的流程已经完成了。如果你在计划类似的应用，希望这篇文章能帮你少走些弯路。
