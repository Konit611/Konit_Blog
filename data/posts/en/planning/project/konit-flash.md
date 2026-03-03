---
title: KonitFlash - Building an Anki-Style Spaced Repetition App with SwiftUI + SwiftData
date: 2026-03-03 19:45
excerpt: SM-2 algorithm, iCloud sync, VIP architecture, CSV import — the technical decisions and implementation behind an iOS/macOS flashcard app.
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

## Why I Built It

I was creating study materials with NotebookLM, then spending ages getting them into Anki. Export as CSV, fix the format, import. I was spending more time on prep than actual studying.

Anki itself had its issues too. Powerful features, but the mobile UX feels stuck in the 2010s. And if you want to seamlessly continue reviewing cards you made on iPhone over on Mac, the setup is a hassle.

Here's what I wanted:

- **One-click import** from NotebookLM CSV
- Proper **spaced repetition** based on the SM-2 algorithm
- **Automatic sync** between iPhone and Mac via iCloud
- A **clean UI** with a dark theme

Nothing out there checked all the boxes, so I built it myself.

<!-- TODO: 3 app screenshots (Home, Study, Result) -->
<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ko/planning/project/konit-flash-home.jpg" alt="Home Screen" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-study.jpg" alt="Study Screen" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-result.jpg" alt="Result Screen" style="width: 32%; border-radius: 8px;" />
</div>

---

## Tech Stack at a Glance

| Area | Technology |
|------|-----------|
| UI | SwiftUI, `horizontalSizeClass` responsive |
| Data | SwiftData (`@Model`) |
| Architecture | VIP (View-Interactor-Presenter) |
| Algorithm | SM-2 Spaced Repetition |
| Sync | CloudKit (SwiftData auto sync) |
| Localization | String Catalog (en/ko/ja/zh-Hans) |
| Widget | WidgetKit + App Groups |
| Testing | Swift Testing (XCTest compatible) |

Built on SwiftUI + SwiftData, targeting iOS 18.6+ / macOS 15.0+.

---

## Architecture: VIP Pattern

### Why VIP

MVVM is a natural fit for SwiftUI, but ViewModels tend to bloat. When business logic and view mapping logic get mixed in one place, testing gets harder and the code becomes difficult to read.

VIP solves this by **separating into three layers**:

```
View ──── Presenter ──── Interactor
(UI)      (Mapping)      (Business Logic)
```

- **View**: Pure UI. Only binds to the Presenter's `@Published` properties.
- **Presenter**: `ObservableObject`. Transforms domain data into ViewState.
- **Interactor**: Pure business logic. Receives `ModelContext` via DI, performs CRUD, and returns **plain structs**.

### VIP in Actual Code

**Interactor** — Receives ModelContext, queries data, returns plain structs:

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

**Presenter** — Maps Interactor output to ViewState:

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

**View** — Owns the Presenter as `@StateObject`, handles only UI:

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

### Deferred Configure Pattern

In SwiftUI, you can't access `@Environment(\.modelContext)` in a View's `init`. So I create the Presenter without arguments and call `configure(modelContext:)` in `.onAppear` — a **deferred initialization** pattern.

```swift
// ❌ Can't access @Environment in init
init() {
    let interactor = HomeInteractor(modelContext: modelContext) // Compile error
}

// ✅ Inject in onAppear
.onAppear {
    presenter.configure(modelContext: modelContext)
}
```

On re-entry (when `configure` gets called twice), the existing Interactor is reused and only `loadData()` is re-executed.

### No Protocols, Keep It Simple

When there's only one implementation, protocols are unnecessary abstraction. In KonitFlash, I chose to **reference implementations directly** without protocols. For testing, injecting an in-memory `ModelContainer` to create an isolated environment is enough.

---

## Data Layer: SwiftData

### Model Design

Three `@Model` classes form the core:

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

    // SM-2 state fields
    var dueDate: Date = Date()
    var interval: Double = 0         // Days until next review
    var easeFactor: Double = 2.5     // Difficulty coefficient
    var repetitions: Int = 0         // Study repetition count
    var box: Int = 1                 // Leitner box (1~5)
}

@Model final class StudyLog {
    var id: UUID = UUID()
    var card: Card?
    var grade: Int = 0               // 0=again, 1=hard, 2=good, 3=easy
    var studiedAt: Date = Date()
    var elapsedSeconds: Double = 0   // Time spent per card
}
```

**Design points:**

- **Cascade Delete**: Deleting a Deck auto-deletes its Cards, and deleting a Card auto-deletes its StudyLogs
- **Default values on all fields**: Required for CloudKit compatibility. Defaults keep things safe during sync conflicts
- **`deckDescription`**: Avoids collision with Swift's `CustomStringConvertible` `description`

### CloudKit Auto Sync

SwiftData's CloudKit integration is a one-liner:

```swift
let config = ModelConfiguration(
    schema: schema,
    cloudKitDatabase: .automatic  // That's all
)
let container = try ModelContainer(for: schema, configurations: [config])
```

But in practice, there were a few gotchas:

1. **No `unique` constraints** — CloudKit doesn't support unique constraints. You have to handle duplicates at the app level. I didn't know this and added `#Unique` to the model, only to get a crash the moment CloudKit kicked in. That took a while to figure out.
2. **Conflict resolution is Last-Writer-Wins** — No alternative strategy available. You have to accept that a card edited on iPhone might get overwritten by Mac.
3. **Offline-first** — Works normally on local SwiftData and auto-syncs when the network recovers. This is actually a strength.

Understanding these constraints before designing models is crucial to avoiding migration hell later. I set up a **fallback chain** at app launch — CloudKit → Local → In-memory — so the app runs in any environment:

```swift
init() {
    do {
        // Primary: CloudKit
        container = try ModelContainer(for: schema,
            configurations: [ModelConfiguration(cloudKitDatabase: .automatic)])
    } catch {
        do {
            // Fallback: Local
            container = try ModelContainer(for: schema,
                configurations: [ModelConfiguration(cloudKitDatabase: .none)])
        } catch {
            // Last resort: In-memory
            container = try! ModelContainer(for: schema,
                configurations: [ModelConfiguration(isStoredInMemoryOnly: true)])
        }
    }
}
```

---

## SM-2 Spaced Repetition Algorithm

### What is SM-2

SM-2 (SuperMemo 2) is a spaced repetition algorithm developed by Piotr Wozniak in 1987. The core idea is simple:

- **Easy cards** → increase the review interval
- **Hard cards** → decrease the review interval
- **Ease Factor (EF)** — a per-card difficulty coefficient, adjusted with each repetition

### Implementation

`SRSEngine` is implemented as an `enum` with pure functions. No side effects, easy to test.

```swift
enum SRSEngine {
    static let minimumEF: Double = 1.3   // EF floor
    static let maximumInterval: Double = 365  // Max 1 year

    static func compute(
        grade: AnswerGrade,
        currentInterval: Double,
        currentEF: Double,
        currentRepetitions: Int,
        currentBox: Int
    ) -> SRSResult {
        switch grade {
        case .again:
            // Full reset: retry in 1 min, repetitions = 0
            newInterval = 1.0 / (24.0 * 60.0)
            newRepetitions = 0

        case .hard:
            // 20% interval increase, EF decrease
            newInterval = max(1, currentInterval * 1.2)
            newEF = max(minimumEF, currentEF - 0.15)

        case .good:
            // Standard SM-2: interval × EF
            if currentRepetitions == 0 { newInterval = 1 }      // 1 day
            else if currentRepetitions == 1 { newInterval = 6 }  // 6 days
            else { newInterval = currentInterval * currentEF }

        case .easy:
            // SM-2 + 30% bonus, EF increase
            newInterval = currentInterval * currentEF * 1.3
            newEF = min(3.0, currentEF + 0.15)
        }
    }
}
```

### Study Session Logic

It's not just showing cards and grading them — it manages a **Learning Queue**:

```
[Review Queue] ─── Cards due for normal review
[Learning Queue] ── Cards waiting for re-study after Again/Hard (based on readyAt time)
```

1. If a card in the Learning Queue has `readyAt <= now`, present it first
2. Otherwise, present the next card from the Review Queue
3. If both are empty but Learning Queue has waiting cards, return a **waiting state**
4. When everything is done, navigate to the result screen

Pressing "Again" makes the card reappear after 1 minute, allowing repeated practice of missed cards within a session. This flow was inspired by Anki's learning steps model.

### Preview Intervals

Answer buttons display the expected review interval so users can make informed decisions:

```
[Again: <1min] [Hard: 1d] [Good: 6d] [Easy: 12d]
```

`SRSEngine.previewIntervals()` computes the expected interval for each of the 4 grades and returns them as strings.

<!-- TODO: Study screen button area screenshot -->

---

## CSV Import: NotebookLM Integration

Export study materials from Google NotebookLM as CSV/TSV, and import them directly into KonitFlash.

### CSVParser Implementation

RFC 4180 compliant + handles real-world edge cases:

```swift
enum CSVParser {
    static func parse(_ content: String) -> CSVParseResult {
        // 1. Strip UTF-8 BOM (common in Excel exports)
        // 2. Auto-detect delimiter (tab vs comma)
        // 3. Auto-skip header row ("Term", "Definition", etc.)
        // 4. RFC 4180 quote escaping
        // 5. Skip empty rows + warn on column count mismatch
    }
}
```

**Auto delimiter detection**: Compares tab and comma frequency in the first line. NotebookLM typically uses TSV, Excel uses CSV, so both need to be supported.

**Header detection**: Detects common header keywords like "Term", "Definition", "Front", "Back", "Question", "Answer" and auto-skips the first row.

### Import Flow

```
File selection (.fileImporter)
    ↓
CSVParser.parse() → Preview (first 5 cards + total N cards)
    ↓
Confirm → Interactor.importCards() → ModelContext save
```

The preview stage shows parsing errors and skipped row counts, letting users identify issues and decide whether to proceed.

<!-- TODO: CSV import preview screen screenshot -->

---

## Responsive UI: One Codebase for iPhone and Mac

### Branching with `horizontalSizeClass`

Using SwiftUI's `horizontalSizeClass`, every screen adapts to iPhone and Mac:

```swift
@Environment(\.horizontalSizeClass) private var sizeClass
private var isRegular: Bool { sizeClass == .regular }

// Font size
.font(.system(size: isRegular ? 64 : 36, weight: .bold))

// Layout
if isRegular {
    LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: 3)) {
        ForEach(decks) { deck in DeckCardView(deck: deck) }
    }
} else {
    ForEach(decks) { deck in DeckCardView(deck: deck) }
}

// Padding
.padding(.horizontal, isRegular ? 40 : 15)
```

In `ContentView`, I manually set `compact`/`regular` based on `GeometryReader`'s 1000pt width threshold. The reason for controlling it directly instead of using system defaults is to precisely control behavior on iPad.

<!-- TODO: iPhone / Mac comparison screenshots side by side -->

### Mac-Specific Considerations

On Mac, keyboard shortcuts were added:

- **Space**: Flip card
- **1~4**: Again / Hard / Good / Easy grade selection

`#if os(macOS)` isolates platform-specific code.

---

## Design System

### Color Tokens

The design uses accent colors on a dark background (`#040422`):

| Token | Color | Usage |
|-------|-------|-------|
| `appBackground` | `#040422` | Full background |
| `overdueText` | `#EF3E3E` | Overdue banner, Again button |
| `streakPink` | `#FFC7EA` | Streak, Hard button, progress bar |
| `learnedGreen` | `#D4F849` | Learned, Good button, Start button |
| `weeklyMint` | `#9CF2E8` | Weekly chart, Easy button |

A Color Extension supports hex initialization, and a `ColorTag` enum maps per-deck colors.

<!-- TODO: Design system color tokens actual screen screenshot -->

### Reusable Components

15 shared components are organized in `DesignSystem/Components/`. Here are a few key patterns.

**3D Card Flip:**

```swift
.rotation3DEffect(.degrees(isFlipped ? 180 : 0), axis: (x: 0, y: 1, z: 0))
.animation(.easeInOut(duration: 0.4), value: isFlipped)
```

Front and back faces toggle with `opacity`, creating a smooth transition during the flip.

**Preventing Layout Jumps:**

Conditionally adding/removing elements with `if` causes layout jumps. Instead, always render but control with `opacity` and `allowsHitTesting`:

```swift
AnswerButtonRow(intervals: intervals, onSelect: onGrade)
    .opacity(isFlipped ? 1 : 0)
    .allowsHitTesting(isFlipped)
```

**Press Feedback:**

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

## Localization

### Runtime Language Switching

Users can change the language within the app, independent of the system language. `LanguageManager` publishes the current `Bundle` via `@Published`, and Presenters subscribe to it to automatically refresh ViewState on language change.

```swift
// Subscribing to language changes in Presenter
LanguageManager.shared.$locale
    .dropFirst()
    .receive(on: RunLoop.main)
    .sink { [weak self] _ in self?.loadData() }
    .store(in: &cancellables)
```

Using String Catalog (`.xcstrings`), 4 languages are supported: en, ko, ja, zh-Hans.

---

## Testing Strategy

Inspired by Vladimir Khorikov's **Unit Testing Principles, Practices, and Patterns**.

### Core Principles

- Test **observable behavior**, not internal implementation
- **Black-box testing**: verify only input → output
- **Mock only shared dependencies**: isolate DB with in-memory `ModelContainer`

### Test Priority

1. **SRSEngine** — The most critical business logic. Verify interval/EF calculations for all grade combinations
2. **CSVParser** — Normal/malformed CSV, BOM, quote escaping
3. **Interactor** — CRUD operations (in-memory ModelContainer)
4. **Presenter** — ViewState mapping verification

```swift
// SRSEngine test example
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
    #expect(result.interval < 0.001)  // ~1 min
}
```

SwiftData tests use in-memory `ModelContainer` to guarantee isolation between tests:

```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)
let container = try ModelContainer(for: Deck.self, Card.self, StudyLog.self,
                                    configurations: config)
let context = ModelContext(container)
let interactor = HomeInteractor(modelContext: context)
```

---

## Tips from the Trenches

### Only One `navigationDestination(isPresented:)`

Using multiple `navigationDestination(isPresented:)` in a single View triggers iOS navigation bugs. The fix is to branch with a local enum:

```swift
enum DetailNavTarget {
    case flashCard(UUID)
    case editCard(UUID, UUID)
}

@State private var navTarget: DetailNavTarget?
@State private var showNav = false

// Handle multiple destinations with a single navigationDestination
.navigationDestination(isPresented: $showNav) {
    switch navTarget {
    case .flashCard(let id): FlashCardView(deckID: id)
    case .editCard(let deckID, let cardID): AddCardView(deckID: deckID, editingCardID: cardID)
    case nil: EmptyView()
    }
}
```

### Reusing Edit Mode

"Add" and "Edit" share nearly identical UI. Instead of creating separate screens, reuse a single View:

```swift
struct AddDeckView: View {
    let editingDeckID: UUID?  // nil = create, UUID = edit

    init(editingDeckID: UUID? = nil) {
        self.editingDeckID = editingDeckID
    }
}
```

The Presenter checks `isEditMode` to dynamically change the header title and button text.

### Debounced Search with Combine

Running a query on every keystroke causes performance issues. Combine's `debounce` adds a 200ms delay:

```swift
$searchText
    .dropFirst()
    .debounce(for: .milliseconds(200), scheduler: RunLoop.main)
    .sink { [weak self] _ in self?.applySearch() }
    .store(in: &cancellables)
```

### Widget Data Sync

WidgetKit runs in a separate process from the app. Data is shared through App Groups' `UserDefaults`, and the widget timeline is refreshed every time the app comes to the foreground.

---

## Retrospective — Being Honest

### Project by the Numbers

| Item | Count |
|------|-------|
| Swift files | 62 |
| Lines of code | ~8,000 |
| Screens (Scenes) | 8 |
| Reusable components | 15 |
| Supported languages | 4 (en, ko, ja, zh-Hans) |
| Test suites | 6 |
| Development period | Phase 0~5 (UI prototype → Widget/Testing) |

### What Went Well

**Applying VIP lightly, without protocols.** In Bullet Journal, I used protocol-based DI, but for a solo project it was overhead. This time, direct implementation references + in-memory ModelContainer testing kept things lean, with no impact on testability.

**Isolating SM-2 as a pure function enum.** The algorithm is the most critical business logic in the entire app. Making it a side-effect-free structure where I only need to test input → output gave me confidence to modify it freely.

**The CSV import UX.** With auto delimiter detection, header skipping, and preview, it all comes down to two taps: "select file → confirm." That's exactly the flow I originally wanted.

### What Could Be Better

**Discovering CloudKit constraints too late.** I learned `unique` constraints don't work only after the model was designed. If I'd run a PoC on CloudKit sync first, the model structure might have been different. Yet another reminder to validate core features first when using new frameworks.

**Test coverage skewed toward SRSEngine and CSVParser.** Interactor and Presenter tests could structurally be written, but I kept pushing them back while chasing feature implementation. The irony of "building a testable structure and then not writing tests" is the same as with Bullet Journal.

**Localization string review.** Four languages are supported, but Japanese and Chinese haven't been reviewed by native speakers. They work functionally, but naturalness of expression needs verification.

---

## Wrapping Up

The biggest lesson from building this app is that **"constraints determine design."**

CloudKit's lack of `unique` support changed the model design. SwiftUI's `@Environment` access restriction created the Deferred Configure pattern. iOS's `navigationDestination` bug forced enum-based routing. None of the original designs survived. Every one of them was a result of hitting constraints.

It's not a perfect app. More tests need to be written, and localization review is still pending. But the original flow I wanted — "export CSV from NotebookLM, one-click import, review with SM-2" — is complete. If you're planning a similar app, I hope this post helps you avoid some of the pitfalls.
