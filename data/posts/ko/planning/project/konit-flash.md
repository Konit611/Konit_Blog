---
title: KonitFlash - SwiftUI + SwiftData로 Anki 스타일 간격 반복 앱 만들기
date: 2026-03-03 19:45
excerpt: SM-2 알고리즘, iCloud 동기화, VIP 아키텍처, CSV 임포트까지 — iOS/macOS 플래시카드 앱을 만들면서 겪은 기술 선택과 구현 과정.
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

## 왜 만들었나

NotebookLM으로 학습 자료를 만들고, 그걸 Anki에 옮기는 데 매번 시간이 걸렸다. CSV로 내보내고, 포맷 맞추고, 임포트하고. 학습보다 준비에 더 많은 시간을 쓰고 있었다.

Anki 자체도 불만이 있었다. 기능은 강력하지만 모바일 UX가 2010년대에 멈춰 있고, iPhone에서 만든 카드를 Mac에서 바로 이어보려면 설정이 번거롭다.

내가 원하는 건 이런 거였다:

- NotebookLM CSV를 **원클릭으로 임포트**할 수 있는 앱
- SM-2 알고리즘 기반 **간격 반복**이 제대로 동작하는 앱
- iCloud로 iPhone ↔ Mac이 **자동 동기화**되는 앱
- 다크 테마 기반의 **깔끔한 UI**

마땅한 게 없어서 직접 만들기로 했다.

<!-- TODO: 앱 스크린샷 3장 (홈, 학습, 결과) -->
<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ko/planning/project/konit-flash-home.jpg" alt="홈 화면" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-study.jpg" alt="학습 화면" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-result.jpg" alt="결과 화면" style="width: 32%; border-radius: 8px;" />
</div>

---

## 기술 스택 한눈에 보기

| 영역 | 기술 |
|------|------|
| UI | SwiftUI, `horizontalSizeClass` 반응형 |
| 데이터 | SwiftData (`@Model`) |
| 아키텍처 | VIP (View-Interactor-Presenter) |
| 알고리즘 | SM-2 Spaced Repetition |
| 동기화 | CloudKit (SwiftData 자동 동기화) |
| 다국어 | String Catalog (en/ko/ja/zh-Hans) |
| 위젯 | WidgetKit + App Groups |
| 테스트 | Swift Testing (XCTest 호환) |

SwiftUI + SwiftData 기반으로, iOS 18.6+ / macOS 15.0+을 타겟한다.

---

## 아키텍처: VIP 패턴

### 왜 VIP인가

MVVM은 SwiftUI와 자연스럽게 맞지만, ViewModel이 비대해지기 쉽다. 비즈니스 로직과 뷰 매핑 로직이 한 곳에 섞이면 테스트도 어려워지고 코드를 읽기도 힘들어진다.

VIP는 이 문제를 **세 레이어로 분리**해서 해결한다:

```
View ──── Presenter ──── Interactor
(UI)      (매핑)         (비즈니스 로직)
```

- **View**: 순수 UI. Presenter의 `@Published` 프로퍼티를 바인딩만 한다.
- **Presenter**: `ObservableObject`. 도메인 데이터를 ViewState로 변환한다.
- **Interactor**: 순수 비즈니스 로직. `ModelContext`를 DI로 받아 CRUD를 수행하고, **plain struct**를 반환한다.

### 실제 코드로 보는 VIP

**Interactor** — ModelContext를 받아 데이터를 조회하고 plain struct로 반환:

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

**Presenter** — Interactor의 출력을 ViewState로 매핑:

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

**View** — Presenter를 `@StateObject`로 소유하고 UI만 담당:

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

### Deferred Configure 패턴

SwiftUI에서는 `@Environment(\.modelContext)`를 View의 `init`에서 접근할 수 없다. 그래서 Presenter를 인자 없이 생성하고, `.onAppear`에서 `configure(modelContext:)`를 호출하는 **지연 초기화** 패턴을 사용한다.

```swift
// ❌ init에서는 @Environment 접근 불가
init() {
    let interactor = HomeInteractor(modelContext: modelContext) // 컴파일 에러
}

// ✅ onAppear에서 주입
.onAppear {
    presenter.configure(modelContext: modelContext)
}
```

재진입 시(`configure`가 두 번 호출되는 경우)에는 기존 Interactor를 재사용하고 `loadData()`만 재실행한다.

### 프로토콜 없이 간결하게

구현체가 하나뿐인 경우 프로토콜은 불필요한 추상화다. KonitFlash에서는 **프로토콜 없이** 직접 참조하는 방식을 택했다. 테스트에서는 in-memory `ModelContainer`를 주입해서 격리된 환경을 만들면 충분하다.

---

## 데이터 레이어: SwiftData

### 모델 설계

세 개의 `@Model` 클래스가 핵심이다:

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

    // SM-2 상태 필드
    var dueDate: Date = Date()
    var interval: Double = 0         // 다음 복습까지 일수
    var easeFactor: Double = 2.5     // 난이도 계수
    var repetitions: Int = 0         // 학습 반복 횟수
    var box: Int = 1                 // Leitner 박스 (1~5)
}

@Model final class StudyLog {
    var id: UUID = UUID()
    var card: Card?
    var grade: Int = 0               // 0=again, 1=hard, 2=good, 3=easy
    var studiedAt: Date = Date()
    var elapsedSeconds: Double = 0   // 카드별 소요 시간
}
```

**설계 포인트:**

- **Cascade Delete**: Deck 삭제 시 하위 Card가 자동 삭제되고, Card 삭제 시 StudyLog도 함께 삭제
- **모든 필드에 기본값**: CloudKit 호환을 위해 필수. 기본값이 있어야 동기화 충돌 시 안전하다
- **`deckDescription`**: `description`은 Swift의 `CustomStringConvertible`과 충돌하므로 회피

### CloudKit 자동 동기화

SwiftData의 CloudKit 통합은 한 줄로 가능하다:

```swift
let config = ModelConfiguration(
    schema: schema,
    cloudKitDatabase: .automatic  // 이 한 줄이 전부
)
let container = try ModelContainer(for: schema, configurations: [config])
```

하지만 실전에서는 몇 가지 함정이 있었다:

1. **`unique` 제약 미지원** — CloudKit은 unique constraint를 지원하지 않는다. 앱 레벨에서 중복을 처리해야 한다. 이걸 모르고 모델에 `#Unique`를 넣었다가 CloudKit 활성화 순간 크래시가 나서 한참 헤맸다.
2. **충돌 해결은 Last-Writer-Wins** — 별도 전략을 선택할 수 없다. iPhone에서 수정한 카드가 Mac에서 덮어써지는 경우를 감수해야 한다.
3. **오프라인 우선** — 로컬 SwiftData에서 정상 동작하고, 네트워크 복구 시 자동 동기화된다. 이건 장점이다.

이런 제약을 미리 파악하고 모델을 설계해야 나중에 마이그레이션 지옥을 피할 수 있다. 앱 시작 시 CloudKit → 로컬 → 인메모리 순으로 **폴백 체인**을 구성해서 어떤 환경에서든 앱이 실행되도록 했다:

```swift
init() {
    do {
        // 1차: CloudKit
        container = try ModelContainer(for: schema,
            configurations: [ModelConfiguration(cloudKitDatabase: .automatic)])
    } catch {
        do {
            // 2차: 로컬
            container = try ModelContainer(for: schema,
                configurations: [ModelConfiguration(cloudKitDatabase: .none)])
        } catch {
            // 3차: 인메모리 (최후의 수단)
            container = try! ModelContainer(for: schema,
                configurations: [ModelConfiguration(isStoredInMemoryOnly: true)])
        }
    }
}
```

---

## SM-2 간격 반복 알고리즘

### SM-2란

SM-2(SuperMemo 2)는 1987년 Piotr Wozniak이 개발한 간격 반복 알고리즘이다. 핵심 아이디어는 단순하다:

- **쉬운 카드** → 복습 간격을 늘린다
- **어려운 카드** → 복습 간격을 줄인다
- **Ease Factor(EF)** — 카드별 난이도 계수. 반복할수록 조정된다

### 구현

`SRSEngine`은 순수 함수로 구현한 `enum`이다. 부작용이 없어 테스트하기 쉽다.

```swift
enum SRSEngine {
    static let minimumEF: Double = 1.3   // EF 하한선
    static let maximumInterval: Double = 365  // 최대 1년

    static func compute(
        grade: AnswerGrade,
        currentInterval: Double,
        currentEF: Double,
        currentRepetitions: Int,
        currentBox: Int
    ) -> SRSResult {
        switch grade {
        case .again:
            // 완전 리셋: 1분 후 재시도, repetitions = 0
            newInterval = 1.0 / (24.0 * 60.0)
            newRepetitions = 0

        case .hard:
            // 간격 20% 증가, EF 감소
            newInterval = max(1, currentInterval * 1.2)
            newEF = max(minimumEF, currentEF - 0.15)

        case .good:
            // 표준 SM-2: interval × EF
            if currentRepetitions == 0 { newInterval = 1 }      // 1일
            else if currentRepetitions == 1 { newInterval = 6 }  // 6일
            else { newInterval = currentInterval * currentEF }

        case .easy:
            // SM-2 + 30% 보너스, EF 증가
            newInterval = currentInterval * currentEF * 1.3
            newEF = min(3.0, currentEF + 0.15)
        }
    }
}
```

### 학습 세션 로직

단순히 카드를 보여주고 등급을 매기는 것이 아니라, **학습 큐(Learning Queue)**를 관리한다:

```
[Review Queue] ─── 정상 복습 대상 카드
[Learning Queue] ── Again/Hard 응답 후 재학습 대기 카드 (readyAt 시간 기준)
```

1. Learning Queue에서 `readyAt <= now`인 카드가 있으면 우선 제시
2. 없으면 Review Queue에서 다음 카드 제시
3. 둘 다 비었지만 Learning Queue에 대기 중인 카드가 있으면 **대기 상태** 반환
4. 모두 완료되면 결과 화면으로 이동

"Again"을 누르면 1분 후에 다시 나타나고, 세션 안에서 틀린 카드를 반복할 수 있다. 이 흐름은 Anki의 학습 스텝 모델에서 영감을 받았다.

### 미리보기 인터벌

답변 버튼에 예상 복습 간격을 표시해서 사용자가 판단할 수 있게 했다:

```
[Again: <1분] [Hard: 1일] [Good: 6일] [Easy: 12일]
```

`SRSEngine.previewIntervals()`가 4개 등급 각각의 예상 간격을 계산해서 문자열로 반환한다.

<!-- TODO: 학습 화면 버튼 영역 스크린샷 -->

---

## CSV 임포트: NotebookLM 연동

Google NotebookLM에서 생성한 학습 자료를 CSV/TSV로 내보내면, KonitFlash에서 바로 가져올 수 있다.

### CSVParser 구현

RFC 4180 호환 + 실전에서 마주치는 엣지 케이스를 처리한다:

```swift
enum CSVParser {
    static func parse(_ content: String) -> CSVParseResult {
        // 1. UTF-8 BOM 제거 (Excel에서 흔함)
        // 2. 구분자 자동 감지 (탭 vs 쉼표)
        // 3. 헤더 행 자동 스킵 ("Term", "Definition" 등)
        // 4. RFC 4180 따옴표 이스케이프 처리
        // 5. 빈 행 스킵 + 열 수 불일치 경고
    }
}
```

**자동 구분자 감지**: 첫 줄에서 탭과 쉼표의 출현 빈도를 비교한다. NotebookLM은 TSV, Excel은 CSV를 주로 사용하므로 둘 다 대응해야 한다.

**헤더 감지**: "Term", "Definition", "Front", "Back", "Question", "Answer" 등 흔한 헤더 키워드를 감지해서 자동으로 첫 행을 스킵한다.

### 임포트 플로우

```
파일 선택 (.fileImporter)
    ↓
CSVParser.parse() → 미리보기 (처음 5장 + 총 N장)
    ↓
확인 → Interactor.importCards() → ModelContext 저장
```

미리보기 단계에서 파싱 에러와 스킵된 행 수를 보여주므로, 사용자가 문제를 인지하고 진행 여부를 결정할 수 있다.

<!-- TODO: CSV 임포트 미리보기 화면 스크린샷 -->

---

## 반응형 UI: iPhone과 Mac을 하나의 코드로

### `horizontalSizeClass`로 분기

SwiftUI의 `horizontalSizeClass`를 활용해 모든 화면이 iPhone과 Mac에 적응한다:

```swift
@Environment(\.horizontalSizeClass) private var sizeClass
private var isRegular: Bool { sizeClass == .regular }

// 폰트 크기
.font(.system(size: isRegular ? 64 : 36, weight: .bold))

// 레이아웃
if isRegular {
    LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: 3)) {
        ForEach(decks) { deck in DeckCardView(deck: deck) }
    }
} else {
    ForEach(decks) { deck in DeckCardView(deck: deck) }
}

// 패딩
.padding(.horizontal, isRegular ? 40 : 15)
```

`ContentView`에서 `GeometryReader`의 너비 1000pt를 기준으로 `compact`/`regular`을 직접 설정한다. 시스템 기본값 대신 직접 제어하는 이유는, iPad에서의 동작을 정밀하게 컨트롤하기 위해서다.

<!-- TODO: iPhone / Mac 비교 스크린샷 나란히 -->

### Mac 전용 고려사항

Mac에서는 키보드 단축키를 추가했다:

- **Space**: 카드 플립
- **1~4**: Again / Hard / Good / Easy 등급 선택

`#if os(macOS)`로 분기해서 플랫폼별 코드를 격리한다.

---

## 디자인 시스템

### 컬러 토큰

다크 배경(`#040422`) 위에 포인트 컬러를 사용하는 디자인이다:

| 토큰 | 색상 | 용도 |
|------|------|------|
| `appBackground` | `#040422` | 전체 배경 |
| `overdueText` | `#EF3E3E` | Overdue 배너, Again 버튼 |
| `streakPink` | `#FFC7EA` | Streak, Hard 버튼, 진행률 바 |
| `learnedGreen` | `#D4F849` | Learned, Good 버튼, Start 버튼 |
| `weeklyMint` | `#9CF2E8` | Weekly 차트, Easy 버튼 |

Color Extension으로 hex 초기화를 지원하고, `ColorTag` enum에서 덱별 색상을 매핑한다.

<!-- TODO: 디자인 시스템 컬러 토큰 실제 화면 스크린샷 -->

### 재사용 컴포넌트

15개의 공통 컴포넌트를 `DesignSystem/Components/`에 모았다. 몇 가지 핵심 패턴을 소개한다.

**3D 카드 플립:**

```swift
.rotation3DEffect(.degrees(isFlipped ? 180 : 0), axis: (x: 0, y: 1, z: 0))
.animation(.easeInOut(duration: 0.4), value: isFlipped)
```

앞면과 뒷면을 각각 `opacity`로 토글해서, 플립 도중 두 면이 자연스럽게 전환된다.

**레이아웃 점프 방지:**

조건부 요소는 `if`로 추가/제거하면 레이아웃이 점프한다. 대신 항상 렌더링하되 `opacity`와 `allowsHitTesting`으로 제어한다:

```swift
AnswerButtonRow(intervals: intervals, onSelect: onGrade)
    .opacity(isFlipped ? 1 : 0)
    .allowsHitTesting(isFlipped)
```

**프레스 피드백:**

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

## 다국어 지원

### 런타임 언어 전환

시스템 언어와 별개로 앱 내에서 언어를 변경할 수 있다. `LanguageManager`가 `@Published`로 현재 `Bundle`을 발행하고, Presenter가 이를 구독해서 언어 변경 시 자동으로 ViewState를 갱신한다.

```swift
// Presenter에서 언어 변경 구독
LanguageManager.shared.$locale
    .dropFirst()
    .receive(on: RunLoop.main)
    .sink { [weak self] _ in self?.loadData() }
    .store(in: &cancellables)
```

String Catalog(`.xcstrings`)를 사용해 en, ko, ja, zh-Hans 4개 언어를 지원한다.

---

## 테스트 전략

Vladimir Khorikov의 **Unit Testing Principles, Practices, and Patterns**에서 영감을 받은 테스트 전략을 적용했다.

### 핵심 원칙

- **관찰 가능한 동작**을 테스트하고, 내부 구현은 테스트하지 않는다
- **블랙박스 테스트**: 입력 → 출력만 검증
- **Mock은 공유 의존성에만**: in-memory `ModelContainer`로 DB를 격리

### 테스트 우선순위

1. **SRSEngine** — 가장 중요한 비즈니스 로직. 모든 등급 조합의 interval/EF 계산 검증
2. **CSVParser** — 정상/비정상 CSV, BOM, 따옴표 이스케이프
3. **Interactor** — CRUD 동작 (in-memory ModelContainer)
4. **Presenter** — ViewState 매핑 검증

```swift
// SRSEngine 테스트 예시
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
    #expect(result.interval < 0.001)  // ~1분
}
```

SwiftData 테스트에서는 in-memory `ModelContainer`를 사용해 테스트 간 격리를 보장한다:

```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)
let container = try ModelContainer(for: Deck.self, Card.self, StudyLog.self,
                                    configurations: config)
let context = ModelContext(container)
let interactor = HomeInteractor(modelContext: context)
```

---

## 실전에서 건진 팁

### `navigationDestination(isPresented:)`는 하나만

한 View에 `navigationDestination(isPresented:)`를 여러 개 사용하면 iOS 네비게이션 버그가 발생한다. 해결법은 로컬 enum으로 분기하는 것이다:

```swift
enum DetailNavTarget {
    case flashCard(UUID)
    case editCard(UUID, UUID)
}

@State private var navTarget: DetailNavTarget?
@State private var showNav = false

// 하나의 navigationDestination으로 여러 목적지 처리
.navigationDestination(isPresented: $showNav) {
    switch navTarget {
    case .flashCard(let id): FlashCardView(deckID: id)
    case .editCard(let deckID, let cardID): AddCardView(deckID: deckID, editingCardID: cardID)
    case nil: EmptyView()
    }
}
```

### 편집 모드 재사용

"추가"와 "편집"은 UI가 거의 동일하다. 별도 화면을 만드는 대신 하나의 View를 재사용한다:

```swift
struct AddDeckView: View {
    let editingDeckID: UUID?  // nil = 생성, UUID = 편집

    init(editingDeckID: UUID? = nil) {
        self.editingDeckID = editingDeckID
    }
}
```

Presenter에서 `isEditMode`를 판단해 헤더 타이틀과 버튼 텍스트를 동적으로 변경한다.

### Debounced Search with Combine

검색어가 바뀔 때마다 쿼리를 실행하면 성능 문제가 생긴다. Combine의 `debounce`로 200ms 지연을 준다:

```swift
$searchText
    .dropFirst()
    .debounce(for: .milliseconds(200), scheduler: RunLoop.main)
    .sink { [weak self] _ in self?.applySearch() }
    .store(in: &cancellables)
```

### 위젯 데이터 동기화

WidgetKit은 앱과 별도 프로세스에서 실행된다. App Groups의 `UserDefaults`를 통해 데이터를 공유하고, 앱이 포그라운드에 올 때마다 위젯 타임라인을 갱신한다.

---

## 회고 — 솔직하게

### 숫자로 보는 프로젝트

| 항목 | 수치 |
|------|------|
| Swift 파일 | 62개 |
| 코드 라인 | ~8,000줄 |
| 화면 (Scene) | 8개 |
| 재사용 컴포넌트 | 15개 |
| 지원 언어 | 4개 (en, ko, ja, zh-Hans) |
| 테스트 스위트 | 6개 |
| 개발 기간 | Phase 0~5 (UI 프로토타입 → 위젯/테스트) |

### 잘한 것

**VIP를 프로토콜 없이 가볍게 적용한 것.** Bullet Journal에서는 프로토콜 기반 DI를 썼는데, 1인 프로젝트에서는 오버헤드였다. 이번에는 구현체 직접 참조 + in-memory ModelContainer 테스트로 간결하게 가져갔고, 테스트에도 문제가 없었다.

**SM-2를 순수 함수 enum으로 격리한 것.** 알고리즘이 앱 전체에서 가장 중요한 비즈니스 로직인데, 부작용 없이 입력 → 출력만 테스트하면 되는 구조로 만들어서 안심하고 수정할 수 있었다.

**CSV 임포트의 사용자 경험.** 자동 구분자 감지, 헤더 스킵, 미리보기까지 넣으니 "파일 선택 → 확인" 두 번의 탭으로 끝난다. 원래 만들고 싶었던 그 흐름이다.

### 아쉬운 것

**CloudKit 제약을 너무 늦게 파악한 것.** `unique` 제약이 안 된다는 걸 모델 설계 후에 알았다. 초반에 PoC로 CloudKit 동기화를 먼저 테스트했으면 모델 구조가 달라졌을 수 있다. 새로운 프레임워크를 쓸 때는 핵심 기능부터 검증하자는 교훈을 또 한번 얻었다.

**테스트 커버리지가 SRSEngine과 CSVParser에 편중된 것.** Interactor와 Presenter 테스트도 구조적으로는 작성할 수 있지만, 기능 구현에 쫓겨 미뤘다. "테스트하기 좋은 구조를 만들어 놓고 테스트를 안 쓴" 아이러니는 Bullet Journal 때와 똑같다.

**다국어 문자열 검수.** 4개 언어를 지원하는데, 일본어와 중국어는 네이티브 검수를 못 했다. 기능적으로 동작하지만 자연스러운 표현인지는 확인이 필요하다.

---

## 마무리

이 앱을 만들면서 가장 크게 배운 건 **"제약이 설계를 결정한다"** 는 것이다.

CloudKit의 `unique` 미지원이 모델 설계를 바꿨고, SwiftUI의 `@Environment` 접근 제한이 Deferred Configure 패턴을 만들었고, iOS의 `navigationDestination` 버그가 enum 기반 라우팅을 강제했다. 처음부터 예상한 설계는 하나도 없었다. 전부 제약에 부딪히면서 나온 결과물이다.

완벽한 앱은 아니다. 테스트도 더 써야 하고, 다국어 검수도 남아 있다. 하지만 "NotebookLM에서 CSV 뽑아서 원클릭으로 임포트하고, SM-2로 복습한다"는 원래 만들고 싶었던 흐름은 완성됐다. 비슷한 앱을 계획하고 있다면, 이 글이 삽질을 줄이는 데 도움이 되면 좋겠다.
