---
title: Bullet Journal - A Retrospective on Applying Clean Architecture to a Personal App and Building 7 Screens
date: 2026-02-22 13:00
excerpt: Architecture decisions, technical challenges, and lessons learned while building a focus timer app with SwiftUI + SwiftData + VIP Architecture.
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

## Why I Built It

The world is overflowing with focus apps. Pomodoro timers, focus modes, growing virtual forests. I've tried them all.

The problem is simple. **You start the timer and open YouTube.** Rather than a matter of willpower, I felt that existing apps just tell you to "stay focused" without actually creating an environment that helps you focus.

Here's what I wanted:

- An app that **plays rain sounds when the timer starts** and lets you control the timer from the lock screen
- An app that **visualizes your daily schedule as a timeline** so you immediately know what you should be doing right now
- An app that lets you **review your weekly focus stats**

Since nothing fit the bill, I decided to build it myself.

<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/en/planning/project/bullet-journal-home_en.jpg" alt="Home" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/en/planning/project/bullet-journal-timeline_en.jpg" alt="Timeline" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/en/planning/project/bullet-journal-dashboard_en.jpg" alt="Dashboard" style="width: 32%; border-radius: 8px;" />
</div>

---

## What I Built

**Bullet Journal** is an iOS app that combines a focus timer + schedule management + statistics dashboard.

| Screen | Role |
|------|------|
| **Home** | Displays tasks for the current time block + starts focus timer |
| **Focus Mode** | Full-screen timer + ambient sounds + pause/stop |
| **Timeline** | Visualizes daily schedule as time blocks, create/edit/delete tasks |
| **Dashboard** | Cumulative focus time + weekly chart + daily records |
| **Daily Log** | Goal completion rate, sleep quality, mood of the day, daily reflection |
| **Settings** | Language switching (KR/EN/JP), app version, privacy policy |
| **Onboarding** | Introduction screen on first launch |

It also supports **lock screen widgets** (Small, Medium, LockScreen) and **lock screen media controls**.

Tech stack:

| Area | Choice |
|------|------|
| UI | SwiftUI (iOS 17+) |
| Data | SwiftData |
| Reactive | Combine |
| Architecture | Clean Architecture (VIP) |
| Concurrency | Swift 6 Strict Concurrency |
| Ads | Google AdMob |
| Language Support | Korean, English, Japanese |

---

## Architecture -- Why I Chose VIP Over MVVM

I initially planned to start with MVVM. It's the most natural pattern for SwiftUI, after all.

But when I listed out all the logic that goes into the home screen alone, I changed my mind:

- Query current time block Tasks from SwiftData
- Start/pause/resume/stop the timer
- Calculate cumulative time from previous sessions + save new sessions
- Play/stop ambient sounds
- Sync with lock screen media controls
- Prompt for sleep quality input
- Refresh widget timelines

If you put all of this into a single ViewModel, you end up with a 500-line God Object. And you can kiss testing goodbye.

VIP structurally separates these concerns:

<!-- VIP architecture diagram placeholder -->
`[Diagram placeholder]`

```
User Action → View → Interactor (Business Logic)
                        ↓ Combine Publisher
                     Presenter (ViewModel Transformation)
                        ↓ @Published
                     View (UI Update)
```

- **View** only draws the UI. Zero logic.
- **Interactor** owns all the business logic. SwiftData queries, timer control, session management.
- **Presenter** transforms domain models into ViewModels for display.

Looking at the actual code, the View receives dependencies, assembles the Interactor, and owns the Presenter as a `@StateObject`:

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

The Presenter subscribes to the Interactor's Combine Publishers and updates `@Published` properties. Note that `receive(on: DispatchQueue.main)` is not used here. Since all services are `@MainActor`, Publishers already emit on the main thread. I originally added it out of habit, but removed it after realizing it caused unnecessary thread hops.

I built all 7 screens with this pattern. **From the 4th screen onward, adding a new screen required almost no deliberation.** You just create Interactor, Presenter, and Models files and fill them in with the same pattern.

---

## The 3 Hardest Challenges

### 1. Background Timer -- Timer Dies in the Background

The most fundamental requirement of a focus timer app turned out to be the trickiest. On iOS, `Timer` stops when the app goes to the background. If a user turns off the screen and comes back 30 minutes later only to find the timer stuck at 0 minutes, the app has no reason to exist.

The solution strategy is to **not rely on Timer at all**. Remember the timer's start time (`startedAt`), and when the app returns to the foreground, recalculate elapsed time as `current time - startedAt`:

```swift
private func handleBackgroundTransition() {
    guard state == .running else { return }
    timer?.invalidate() // Stop the Timer
    timer = nil // But preserve startedAt
}

private func handleForegroundTransition() {
    guard state == .running else { return }
    updateElapsedTime() // Recalculate elapsed time from time difference
    tickSubject.send(elapsedSeconds) // Immediately update UI
    startTimer() // Restart Timer
}
```

Initially, I put the logic inline inside the `appDidEnterBackground` callback, which caused a race condition. The NotificationCenter callback and the Timer callback were executing simultaneously, corrupting `elapsedSeconds`. Extracting the handlers into separate methods fixed the issue and, as a bonus, made it possible to write unit tests.

A small detail that makes a big difference in user experience: **send a tick immediately on resume**. Without this, the screen appears frozen for up to 1 second after pressing the RESUME button. That single line `tickSubject.send(elapsedSeconds)` dramatically improves the UX.

### 2. SwiftData -- Enum Predicates Crash at Runtime

I chose SwiftData because it's more Swift-native than Core Data and the code is cleaner. I was mostly satisfied, but one issue caught me off guard.

**Comparing Enums in a Predicate causes a runtime crash:**

```swift
// Crashes
#Predicate<FocusSession> { $0.status == .completed }

// Still crashes even with a variable
let status = FocusSessionStatus.completed
#Predicate<FocusSession> { $0.status == status }
```

The workaround is straightforward. **Fetch everything and filter in memory:**

```swift
let all = try modelContext.fetch(FetchDescriptor<FocusSession>())
let completed = all.filter { $0.status == .completed }
```

Since this is a personal app and the data never exceeds a few thousand records, there were no performance issues. In fact, the Single Fetch pattern -- fetching once and reusing across multiple places -- actually made the code cleaner.

Lesson learned: **When using a new framework, validate your core query patterns with a PoC first.**

### 3. Ambient Sounds -- Crossfade Is More Complex Than Expected

I added a feature to loop ambient sounds like rain or birdsong during focus mode. Playback itself is simple with `AVAudioPlayer`, but the problem was **sound transitions**.

When the user switches from "rain" to "birdsong," an abrupt cut feels jarring. A smooth crossfade was needed. I implemented logic that fades out the previous sound over 1 second while fading in the new one, which required managing a separate timer that adjusts volume every 0.05 seconds.

Adding lock screen media control integration (`NowPlayingService`) on top of this increased the complexity further. When the user presses pause on the lock screen, both the timer and the sound need to stop. I solved this by grouping these three services (Timer, AmbientSound, NowPlaying) into a `ServiceContainer`, with the Interactor orchestrating them:

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

    // Testing: Mock injection supported
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

In production, it uses `ServiceContainer.shared`; in tests, mocks are injected. It's a singleton, but DI is open, so testing is no problem.

---

## 3 Practical Tips from the Trenches

### Swift 6's @MainActor Is Truly Strict

Code like this triggers a compile error:

```swift
init(localizationManager: LocalizationManager = .shared) { }
```

`LocalizationManager.shared` is `@MainActor`, but default parameters are evaluated at the call site, where a MainActor context isn't guaranteed. At first I was confused -- "Why doesn't this work?" But ultimately, it forced explicit DI, which led to more testable code.

### Data Sharing Between Widgets and the App

Widgets run in a separate process from the app. To access the same SwiftData store, you need to set up a shared container via **App Group**. I unified the path using `AppConfiguration.sharedStoreURL` and call `WidgetCenter.shared.reloadAllTimelines()` whenever a session starts or ends in the app to refresh the widgets. The setup itself is straightforward, but if you miss it, the widget always shows a blank screen, so it's something you need to address early on.

### Localization Is Painful If You Don't Do It from the Start

The app supports 3 languages (Korean, English, Japanese), and managing everything with `.xcstrings` from the beginning saved me a lot of headaches. Had I tried to add localization late in development, I would have had to find and replace hundreds of hardcoded strings.

One gotcha: SwiftUI Alert titles don't get localized if you pass a string literal. You must wrap them with `Text()`. It took me quite a while to discover this.

---

## Retrospective -- Being Honest

### The Project in Numbers

| Item | Count |
|------|------|
| Swift files | 84 (App 68 + Tests 7 + Widgets, etc.) |
| Screens | 7 (all using VIP pattern) |
| Services | 3 (Timer, AmbientSound, NowPlaying) |
| Widgets | 4 types (Small, Medium, LockScreen Circular, LockScreen Rectangular) |
| Supported languages | 3 (Korean, English, Japanese) |

### What Went Well

**Sticking with VIP all the way through.** All 7 screens share the same structure. When adding a new screen, I just follow the same template without any deliberation. "Where does this logic live?" -- always in the Interactor. "I want to change how something is displayed" -- always in the Presenter. It takes 5 seconds to find anything.

**Separating services behind protocols.** TimerService, AmbientSoundService, and NowPlayingService all have protocols. This made it possible to create mocks and test Interactors independently.

**Handling the background timer properly.** By extracting handlers into separate methods and using time-difference calculation, the timer remains accurate even after screen lock and resume.

### What I'd Do Differently

**Not writing enough tests.** The VIP structure makes testing easy, but I was too busy implementing features to achieve good coverage. It's ironic -- "I built a highly testable architecture and then didn't write the tests."

**Trusting SwiftData too much.** I assumed that since it's Apple's first-party framework, everything would just work. Had I discovered the Enum Predicate crash earlier, I might have designed the models differently.

### Is Clean Architecture Overkill for a Personal Project?

The short answer is **no.** It might feel like overkill for a solo project, but its true value becomes clear when you reopen the code two weeks later.

That said, for a simple app with 2-3 screens, MVVM is more than enough. VIP shines when you have many screens and complex business logic. This app has 7 screens with timers, sounds, lock screen integration, and widgets all intertwined, so VIP was the right choice.

---

## Wrapping Up

The biggest lesson from building this app is that **"structure is speed."**

The time spent setting up the architecture early on can feel like a waste. But when you're building the 7th screen, fixing a background timer bug, or needing the widget to read the same data -- that investment pays back many times over.

It's not a perfect app. There's still a lot I want to do. But through the process of building it, I gained hands-on experience with SwiftUI, SwiftData, Combine, and Swift Concurrency at a production level. If you're planning a similar app, I hope this post helps you avoid some of the pitfalls I encountered.
