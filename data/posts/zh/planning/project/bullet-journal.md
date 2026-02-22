---
title: Bullet Journal - 在个人应用中实践 Clean Architecture 并构建 7 个页面的心得
date: 2026-02-22 13:00
excerpt: 使用 SwiftUI + SwiftData + VIP Architecture 开发专注计时器应用过程中的架构决策、技术难点与复盘总结。
coverImage:
categories:
  - project
tags:
  - SwiftUI
  - SwiftData
  - CleanArchitecture
  - iOS
author: Geunil Park
featured: false
---

## 为什么要做这个应用

专注力应用遍地都是。番茄钟、专注模式、种树。我全都用过了。

问题只有一个。**开着计时器却打开了 YouTube。** 与其说是意志力的问题，不如说是因为现有的应用只是告诉你"请专注"，却没有真正为你营造一个有助于专注的环境。

我想要的是这样的应用：

- 启动计时器后**自动播放雨声**，在锁屏界面也能控制计时器的应用
- 将一天的日程以**时间线可视化**，让你一眼就知道现在该做什么的应用
- 能够通过**统计数据回顾**一周专注了多少时间的应用

找不到满意的，于是决定自己动手做。

<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/zh/planning/project/bullet-journal-home_en.jpg" alt="主页" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/zh/planning/project/bullet-journal-timeline_en.jpg" alt="时间线" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/zh/planning/project/bullet-journal-dashboard_en.jpg" alt="仪表盘" style="width: 32%; border-radius: 8px;" />
</div>

---

## 做了什么

**Bullet Journal** 是一款集专注计时器 + 日程管理 + 统计仪表盘于一体的 iOS 应用。

| 页面 | 功能 |
|------|------|
| **主页** | 显示当前时间段的待办事项 + 启动专注计时器 |
| **专注模式** | 全屏计时器 + 环境音效 + 暂停/停止 |
| **时间线** | 将一天的日程以时间块可视化，支持待办事项的创建/编辑/删除 |
| **仪表盘** | 累计专注时长 + 周报图表 + 每日记录 |
| **每日记录** | 目标完成率、睡眠质量、今日心情、复盘写作 |
| **设置** | 语言切换（韩/英/日）、应用版本、隐私政策 |
| **引导页** | 首次启动时的介绍页面 |

此外还支持**锁屏小组件**（Small、Medium、LockScreen）和**锁屏媒体控制**。

技术栈：

| 领域 | 选择 |
|------|------|
| UI | SwiftUI (iOS 17+) |
| 数据 | SwiftData |
| 响应式 | Combine |
| 架构 | Clean Architecture (VIP) |
| 并发 | Swift 6 Strict Concurrency |
| 广告 | Google AdMob |
| 语言支持 | 韩语、英语、日语 |

---

## 架构 — 为什么选择 VIP 而不是 MVVM

一开始我打算用 MVVM。毕竟这是 SwiftUI 中最自然的模式。

但当我列出仅主页一个页面需要处理的逻辑时，想法改变了：

- 从 SwiftData 查询当前时间段的 Task
- 计时器的启动/暂停/恢复/停止
- 计算之前的会话累计时长 + 保存新会话
- 环境音效的播放/停止
- 锁屏媒体控制联动
- 睡眠质量输入提示
- 小组件时间线刷新

如果把这些全塞进一个 ViewModel，就会变成一个 500 行的 God Object。测试也只能放弃。

VIP 从结构上解决了这个问题：

<!-- VIP 架构图插入位置 -->
`[架构图插入位置]`

```
用户操作 → View → Interactor (业务逻辑)
                        ↓ Combine Publisher
                     Presenter (ViewModel 转换)
                        ↓ @Published
                     View (页面刷新)
```

- **View** 只负责绘制 UI。零逻辑。
- **Interactor** 承载全部业务逻辑。SwiftData 查询、计时器控制、会话管理。
- **Presenter** 将领域模型转换为页面展示用的 ViewModel。

看看实际代码，View 接收依赖并组装 Interactor，将 Presenter 作为 `@StateObject` 持有：

```swift
struct HomeView: View {
    @StateObject private var presenter: HomePresenter

    init(modelContext: ModelContext, serviceContainer: ServiceContainer) {
        let interactor = HomeInteractor(
            modelContext: modelContext,
            timerService: serviceContainer.timerService,
            ambientSoundService: serviceContainer.ambientSoundService,
            nowPlayingService: serviceContainer.nowPlayingService
        )
        _presenter = StateObject(wrappedValue: HomePresenter(interactor: interactor))
    }
}
```

Presenter 订阅 Interactor 的 Combine Publisher 来更新 `@Published` 属性。这里不使用 `receive(on: DispatchQueue.main)`。因为所有服务都标记了 `@MainActor`，Publisher 本身就在主线程上发布。一开始我出于习惯加上了，后来发现会产生不必要的线程跳转，就移除了。

用这个模式构建了全部 7 个页面。**从第 4 个页面开始，添加新页面几乎不需要思考。** 创建 Interactor、Presenter、Models 文件，按照相同的模式填充即可。

---

## 最困难的 3 件事

### 1. 后台计时器 — Timer 在后台会被终止

专注计时器应用最基本的需求，却是最棘手的问题。在 iOS 中，`Timer` 在应用进入后台后会停止。用户关掉屏幕 30 分钟后回来，发现计时器还停在 0 分钟，那这个应用就失去了存在的意义。

解决策略是**不依赖 Timer**。记住计时器启动时刻（`startedAt`），在回到前台时用 `当前时刻 - startedAt` 重新计算经过的时间：

```swift
private func handleBackgroundTransition() {
    guard state == .running else { return }
    timer?.invalidate() // Timer 정지
    timer = nil // 하지만 startedAt은 보존
}

private func handleForegroundTransition() {
    guard state == .running else { return }
    updateElapsedTime() // 시간 차이로 경과 시간 재계산
    tickSubject.send(elapsedSeconds) // 즉시 UI 업데이트
    startTimer() // Timer 재시작
}
```

一开始我把逻辑内联写在 `appDidEnterBackground` 回调里，结果发生了 Race Condition。NotificationCenter 回调和 Timer 回调同时执行，导致 `elapsedSeconds` 出现混乱。将处理逻辑抽离为独立方法后问题解决了，还附带获得了编写单元测试的能力。

一个看似微小但体感明显的细节。Resume 时**立即发送 tick**。如果不这样做，按下 RESUME 按钮后页面最多会卡顿 1 秒。`tickSubject.send(elapsedSeconds)` 这一行代码就能显著改善用户体验。

### 2. SwiftData — Enum Predicate 会导致崩溃

选择 SwiftData 是因为它比 Core Data 更加 Swift 原生，代码也更简洁。大部分体验令人满意，但有一个问题让我措手不及。

**在 Predicate 中比较 Enum 会导致运行时崩溃：**

```swift
// 크래시
#Predicate<FocusSession> { $0.status == .completed }

// 변수에 담아도 크래시
let status = FocusSessionStatus.completed
#Predicate<FocusSession> { $0.status == status }
```

最终的解决方法很简单。**全部取出后在内存中过滤：**

```swift
let all = try modelContext.fetch(FetchDescriptor<FocusSession>())
let completed = all.filter { $0.status == .completed }
```

因为是个人应用，数据量不会超过几千条，所以没有性能问题。反而一次 fetch 后在多处复用的 Single Fetch 模式让代码更加简洁。

教训：**使用新框架时，应先用 PoC 验证核心查询模式。**

### 3. 环境音效 — 交叉淡入淡出比想象中复杂

在专注模式中加入了雨声、鸟鸣等环境音效的循环播放功能。播放本身用 `AVAudioPlayer` 很简单，问题在于**音效切换**。

用户从"雨声"切换到"鸟鸣"时如果声音突然中断，体验会很差。需要自然的交叉淡入淡出。我实现了在 1 秒内淡出旧音效同时淡入新音效的逻辑，但需要额外管理一个每 0.05 秒调整音量的定时器。

再加上锁屏媒体控制（`NowPlayingService`）的联动，复杂度进一步上升。在锁屏界面按暂停，计时器和音效都要停止。我将这三个服务（Timer、AmbientSound、NowPlaying）的状态统一封装在 `ServiceContainer` 中，由 Interactor 进行编排：

```swift
@MainActor
final class ServiceContainer {
    static let shared = ServiceContainer()

    let timerService: TimerServiceProtocol
    let ambientSoundService: AmbientSoundServiceProtocol
    let nowPlayingService: NowPlayingServiceProtocol

    private init() {
        self.timerService = TimerService()
        self.ambientSoundService = AmbientSoundService()
        self.nowPlayingService = NowPlayingService()
    }

    // 테스트: Mock 주입 가능
    init(
        timerService: TimerServiceProtocol,
        ambientSoundService: AmbientSoundServiceProtocol,
        nowPlayingService: NowPlayingServiceProtocol
    ) {
        self.timerService = timerService
        self.ambientSoundService = ambientSoundService
        self.nowPlayingService = nowPlayingService
    }
}
```

生产环境使用 `ServiceContainer.shared`，测试时注入 Mock。虽然是单例，但 DI 接口是开放的，因此不影响测试。

---

## 实战中总结的 3 个技巧

### Swift 6 的 @MainActor 真的很严格

这样的代码会产生编译错误：

```swift
init(localizationManager: LocalizationManager = .shared) { }
```

因为 `LocalizationManager.shared` 标记了 `@MainActor`，而默认参数在调用时才求值，无法保证处于 MainActor 上下文中。一开始我很困惑"这为什么不行？"，但结果是它强制使用显式 DI，反而写出了更易测试的代码。

### 小组件与应用之间的数据共享

小组件在独立于应用的进程中运行。要访问同一个 SwiftData，需要通过 **App Group** 设置共享容器。使用 `AppConfiguration.sharedStoreURL` 统一路径，每当应用中的会话开始/结束时调用 `WidgetCenter.shared.reloadAllTimelines()` 来刷新小组件。设置本身很简单，但如果遗漏了这一步，小组件就会一直显示空白页面，所以这是初期就要处理好的部分。

### 多语言支持如果不从一开始做，后面会很痛苦

应用支持 3 种语言（韩语、英语、日语），因为从一开始就用 `.xcstrings` 管理，所以没遇到太大困难。如果打算在开发后期才加入多语言支持，就得找到并替换数百个硬编码的字符串。

有一个坑需要注意。SwiftUI Alert 的 title 如果直接传入字符串字面量，本地化不会生效。必须用 `Text()` 包裹。发现这个问题花了我不少时间。

---

## 复盘 — 坦诚地说

### 用数字看项目

| 项目 | 数值 |
|------|------|
| Swift 文件数 | 84 个（应用 68 + 测试 7 + 小组件等） |
| 页面数 | 7 个（全部采用 VIP 模式） |
| 服务 | 3 个（Timer、AmbientSound、NowPlaying） |
| 小组件 | 4 种（Small、Medium、LockScreen Circular、LockScreen Rectangular） |
| 支持语言 | 3 种（韩语、英语、日语） |

### 做得好的地方

**坚持到底使用了 VIP。** 7 个页面全部采用相同的结构。添加新页面时无需思考，按照相同的模板即可。"这个逻辑在哪里？" —— 一定在 Interactor 里。"想修改显示格式？" —— 一定在 Presenter 里。5 秒就能找到。

**将服务通过协议分离。** TimerService、AmbientSoundService、NowPlayingService 都有对应的协议。因此可以创建 Mock 来独立测试 Interactor。

**正确处理了后台计时器。** 将处理逻辑抽离为独立方法，采用时间差计算方式，确保锁屏后恢复时计时器依然准确。

### 遗憾的地方

**没有写足够的测试。** 得益于 VIP 结构，测试编写本应很容易，但忙于实现功能导致覆盖率偏低。变成了"搭建了易于测试的架构，却没有写测试"的讽刺局面。

**过于信任 SwiftData。** 想着它是 Apple 的第一方框架，应该不会有问题。如果早点发现 Enum Predicate 崩溃的问题，模型设计可能会有所不同。

### 个人项目使用 Clean Architecture 是否过度？

先说结论：**不过度。** 独自开发的应用可能会觉得过于繁重，但当你两周后再次打开代码时，它的价值就会体现出来。

不过如果只有 2~3 个页面的简单应用，MVVM 就足够了。VIP 在页面多且业务逻辑复杂时才能发挥优势。这个应用有 7 个页面，还涉及计时器、音效、锁屏、小组件等多种功能的联动，所以 VIP 是正确的选择。

---

## 结语

开发这个应用最大的收获是理解了**"结构即速度"**这句话的含义。

初期花在搭建架构上的时间会让人觉得可惜。但当你构建第 7 个页面时、修复后台计时器 Bug 时、需要在小组件中读取相同数据时 —— 那份投入会成倍回报。

这不是一个完美的应用。还有很多想做的事情。但在开发过程中，我得以在实战层面深入体验了 SwiftUI、SwiftData、Combine、Swift Concurrency。如果你也在计划类似的应用，希望这篇文章能帮你少走一些弯路。
