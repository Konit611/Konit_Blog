---
title: KonitFlash - SwiftUI + SwiftDataでAnkiスタイルの間隔反復アプリを作る
date: 2026-03-03 19:45
excerpt: SM-2アルゴリズム、iCloud同期、VIPアーキテクチャ、CSVインポートまで — iOS/macOSフラッシュカードアプリの技術選定と実装過程。
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

## なぜ作ったのか

NotebookLMで学習資料を作って、それをAnkiに移すのに毎回時間がかかっていた。CSVでエクスポートして、フォーマットを合わせて、インポートして。学習よりも準備に多くの時間を使っていた。

Anki自体にも不満があった。機能は強力だが、モバイルUXが2010年代で止まっている。iPhoneで作ったカードをMacですぐに続けて見ようとすると、設定が面倒だ。

自分が欲しかったのはこういうものだった：

- NotebookLMのCSVを**ワンクリックでインポート**できるアプリ
- SM-2アルゴリズムベースの**間隔反復**がちゃんと動くアプリ
- iCloudでiPhone ↔ Macが**自動同期**されるアプリ
- ダークテーマベースの**すっきりしたUI**

ぴったりのものがなかったので、自分で作ることにした。

<!-- TODO: アプリスクリーンショット3枚（ホーム、学習、結果） -->
<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ko/planning/project/konit-flash-home.jpg" alt="ホーム画面" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-study.jpg" alt="学習画面" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/konit-flash-result.jpg" alt="結果画面" style="width: 32%; border-radius: 8px;" />
</div>

---

## 技術スタック一覧

| 領域 | 技術 |
|------|------|
| UI | SwiftUI、`horizontalSizeClass`レスポンシブ |
| データ | SwiftData（`@Model`） |
| アーキテクチャ | VIP（View-Interactor-Presenter） |
| アルゴリズム | SM-2 Spaced Repetition |
| 同期 | CloudKit（SwiftData自動同期） |
| 多言語 | String Catalog（en/ko/ja/zh-Hans） |
| ウィジェット | WidgetKit + App Groups |
| テスト | Swift Testing（XCTest互換） |

SwiftUI + SwiftDataベースで、iOS 18.6+ / macOS 15.0+をターゲットとしている。

---

## アーキテクチャ：VIPパターン

### なぜVIPなのか

MVVMはSwiftUIと自然にマッチするが、ViewModelが肥大化しやすい。ビジネスロジックとビューマッピングロジックが一箇所に混ざると、テストも難しくなりコードも読みにくくなる。

VIPはこの問題を**3つのレイヤーに分離**して解決する：

```
View ──── Presenter ──── Interactor
(UI)      (マッピング)    (ビジネスロジック)
```

- **View**：純粋なUI。Presenterの`@Published`プロパティをバインドするだけ。
- **Presenter**：`ObservableObject`。ドメインデータをViewStateに変換する。
- **Interactor**：純粋なビジネスロジック。`ModelContext`をDIで受け取り、CRUDを実行し、**plain struct**を返す。

### 実際のコードで見るVIP

**Interactor** — ModelContextを受け取り、データを取得してplain structで返す：

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

**Presenter** — Interactorの出力をViewStateにマッピング：

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

**View** — Presenterを`@StateObject`として保持し、UIのみを担当：

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

### Deferred Configureパターン

SwiftUIでは`@Environment(\.modelContext)`をViewの`init`でアクセスできない。そのため、Presenterを引数なしで生成し、`.onAppear`で`configure(modelContext:)`を呼び出す**遅延初期化**パターンを使用する。

```swift
// ❌ initでは@Environmentにアクセス不可
init() {
    let interactor = HomeInteractor(modelContext: modelContext) // コンパイルエラー
}

// ✅ onAppearで注入
.onAppear {
    presenter.configure(modelContext: modelContext)
}
```

再入時（`configure`が2回呼ばれる場合）は既存のInteractorを再利用し、`loadData()`のみ再実行する。

### プロトコルなしでシンプルに

実装が一つだけの場合、プロトコルは不要な抽象化だ。KonitFlashでは**プロトコルなし**で直接参照する方式を選んだ。テストではin-memoryの`ModelContainer`を注入して隔離された環境を作れば十分だ。

---

## データレイヤー：SwiftData

### モデル設計

3つの`@Model`クラスがコアだ：

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

    // SM-2状態フィールド
    var dueDate: Date = Date()
    var interval: Double = 0         // 次の復習までの日数
    var easeFactor: Double = 2.5     // 難易度係数
    var repetitions: Int = 0         // 学習反復回数
    var box: Int = 1                 // Leitnerボックス（1〜5）
}

@Model final class StudyLog {
    var id: UUID = UUID()
    var card: Card?
    var grade: Int = 0               // 0=again, 1=hard, 2=good, 3=easy
    var studiedAt: Date = Date()
    var elapsedSeconds: Double = 0   // カードごとの所要時間
}
```

**設計ポイント：**

- **Cascade Delete**：Deck削除時に配下のCardが自動削除され、Card削除時にStudyLogも一緒に削除
- **全フィールドにデフォルト値**：CloudKit互換のために必須。デフォルト値があることで同期衝突時も安全
- **`deckDescription`**：`description`はSwiftの`CustomStringConvertible`と衝突するため回避

### CloudKit自動同期

SwiftDataのCloudKit統合は一行で可能だ：

```swift
let config = ModelConfiguration(
    schema: schema,
    cloudKitDatabase: .automatic  // この一行だけ
)
let container = try ModelContainer(for: schema, configurations: [config])
```

しかし実戦ではいくつかの罠があった：

1. **`unique`制約非対応** — CloudKitはunique constraintをサポートしない。アプリレベルで重複を処理する必要がある。これを知らずにモデルに`#Unique`を入れたら、CloudKit有効化の瞬間にクラッシュして、かなりハマった。
2. **衝突解決はLast-Writer-Wins** — 別の戦略を選択できない。iPhoneで修正したカードがMacで上書きされるケースを受け入れなければならない。
3. **オフラインファースト** — ローカルSwiftDataで正常動作し、ネットワーク復旧時に自動同期される。これは長所だ。

これらの制約を事前に把握してモデルを設計しないと、後でマイグレーション地獄にハマる。アプリ起動時にCloudKit → ローカル → インメモリの順で**フォールバックチェーン**を構成し、どの環境でもアプリが実行できるようにした：

```swift
init() {
    do {
        // 1次：CloudKit
        container = try ModelContainer(for: schema,
            configurations: [ModelConfiguration(cloudKitDatabase: .automatic)])
    } catch {
        do {
            // 2次：ローカル
            container = try ModelContainer(for: schema,
                configurations: [ModelConfiguration(cloudKitDatabase: .none)])
        } catch {
            // 3次：インメモリ（最後の手段）
            container = try! ModelContainer(for: schema,
                configurations: [ModelConfiguration(isStoredInMemoryOnly: true)])
        }
    }
}
```

---

## SM-2間隔反復アルゴリズム

### SM-2とは

SM-2（SuperMemo 2）は1987年にPiotr Wozniakが開発した間隔反復アルゴリズムだ。コアアイデアはシンプルだ：

- **簡単なカード** → 復習間隔を延ばす
- **難しいカード** → 復習間隔を縮める
- **Ease Factor（EF）** — カードごとの難易度係数。反復するたびに調整される

### 実装

`SRSEngine`は純粋関数として実装した`enum`だ。副作用がないのでテストしやすい。

```swift
enum SRSEngine {
    static let minimumEF: Double = 1.3   // EF下限
    static let maximumInterval: Double = 365  // 最大1年

    static func compute(
        grade: AnswerGrade,
        currentInterval: Double,
        currentEF: Double,
        currentRepetitions: Int,
        currentBox: Int
    ) -> SRSResult {
        switch grade {
        case .again:
            // 完全リセット：1分後にリトライ、repetitions = 0
            newInterval = 1.0 / (24.0 * 60.0)
            newRepetitions = 0

        case .hard:
            // 間隔20%増加、EF減少
            newInterval = max(1, currentInterval * 1.2)
            newEF = max(minimumEF, currentEF - 0.15)

        case .good:
            // 標準SM-2：interval × EF
            if currentRepetitions == 0 { newInterval = 1 }      // 1日
            else if currentRepetitions == 1 { newInterval = 6 }  // 6日
            else { newInterval = currentInterval * currentEF }

        case .easy:
            // SM-2 + 30%ボーナス、EF増加
            newInterval = currentInterval * currentEF * 1.3
            newEF = min(3.0, currentEF + 0.15)
        }
    }
}
```

### 学習セッションロジック

単にカードを見せてグレードを付けるだけでなく、**学習キュー（Learning Queue）**を管理する：

```
[Review Queue] ─── 通常の復習対象カード
[Learning Queue] ── Again/Hard応答後の再学習待機カード（readyAt時間基準）
```

1. Learning Queueで`readyAt <= now`のカードがあれば優先提示
2. なければReview Queueから次のカードを提示
3. 両方空だがLearning Queueに待機中のカードがあれば**待機状態**を返す
4. すべて完了したら結果画面に遷移

「Again」を押すと1分後に再び表示され、セッション内で間違えたカードを繰り返し練習できる。このフローはAnkiの学習ステップモデルからインスピレーションを受けた。

### プレビューインターバル

回答ボタンに予想復習間隔を表示して、ユーザーが判断できるようにした：

```
[Again: <1分] [Hard: 1日] [Good: 6日] [Easy: 12日]
```

`SRSEngine.previewIntervals()`が4つのグレードそれぞれの予想間隔を計算して文字列で返す。

<!-- TODO: 学習画面ボタンエリアのスクリーンショット -->

---

## CSVインポート：NotebookLM連携

Google NotebookLMで生成した学習資料をCSV/TSVでエクスポートすれば、KonitFlashで直接取り込める。

### CSVParser実装

RFC 4180準拠 + 実戦で遭遇するエッジケースに対応：

```swift
enum CSVParser {
    static func parse(_ content: String) -> CSVParseResult {
        // 1. UTF-8 BOM除去（Excelでよくある）
        // 2. デリミタ自動検出（タブ vs カンマ）
        // 3. ヘッダー行自動スキップ（"Term"、"Definition"など）
        // 4. RFC 4180クォートエスケープ処理
        // 5. 空行スキップ + 列数不一致警告
    }
}
```

**自動デリミタ検出**：最初の行でタブとカンマの出現頻度を比較する。NotebookLMはTSV、ExcelはCSVを主に使うので、両方対応が必要だ。

**ヘッダー検出**："Term"、"Definition"、"Front"、"Back"、"Question"、"Answer"など一般的なヘッダーキーワードを検出して、自動的に最初の行をスキップする。

### インポートフロー

```
ファイル選択（.fileImporter）
    ↓
CSVParser.parse() → プレビュー（最初の5枚 + 合計N枚）
    ↓
確認 → Interactor.importCards() → ModelContext保存
```

プレビュー段階でパースエラーとスキップされた行数を表示するので、ユーザーが問題を認識して続行するかどうか判断できる。

<!-- TODO: CSVインポートプレビュー画面のスクリーンショット -->

---

## レスポンシブUI：iPhoneとMacを1つのコードで

### `horizontalSizeClass`で分岐

SwiftUIの`horizontalSizeClass`を活用して、すべての画面がiPhoneとMacに適応する：

```swift
@Environment(\.horizontalSizeClass) private var sizeClass
private var isRegular: Bool { sizeClass == .regular }

// フォントサイズ
.font(.system(size: isRegular ? 64 : 36, weight: .bold))

// レイアウト
if isRegular {
    LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: 3)) {
        ForEach(decks) { deck in DeckCardView(deck: deck) }
    }
} else {
    ForEach(decks) { deck in DeckCardView(deck: deck) }
}

// パディング
.padding(.horizontal, isRegular ? 40 : 15)
```

`ContentView`で`GeometryReader`の幅1000ptを基準に`compact`/`regular`を直接設定する。システムデフォルトではなく直接制御する理由は、iPadでの動作を精密にコントロールするためだ。

<!-- TODO: iPhone / Mac比較スクリーンショット並べて -->

### Mac専用の考慮事項

Macではキーボードショートカットを追加した：

- **Space**：カードフリップ
- **1〜4**：Again / Hard / Good / Easy グレード選択

`#if os(macOS)`で分岐してプラットフォーム別コードを隔離する。

---

## デザインシステム

### カラートークン

ダーク背景（`#040422`）の上にアクセントカラーを使うデザインだ：

| トークン | 色 | 用途 |
|---------|------|------|
| `appBackground` | `#040422` | 全体背景 |
| `overdueText` | `#EF3E3E` | Overdueバナー、Againボタン |
| `streakPink` | `#FFC7EA` | Streak、Hardボタン、プログレスバー |
| `learnedGreen` | `#D4F849` | Learned、Goodボタン、Startボタン |
| `weeklyMint` | `#9CF2E8` | Weeklyチャート、Easyボタン |

Color Extensionでhex初期化をサポートし、`ColorTag` enumでデッキごとの色をマッピングする。

<!-- TODO: デザインシステムカラートークン実画面スクリーンショット -->

### 再利用コンポーネント

15個の共通コンポーネントを`DesignSystem/Components/`にまとめた。いくつかの重要なパターンを紹介する。

**3Dカードフリップ：**

```swift
.rotation3DEffect(.degrees(isFlipped ? 180 : 0), axis: (x: 0, y: 1, z: 0))
.animation(.easeInOut(duration: 0.4), value: isFlipped)
```

表面と裏面をそれぞれ`opacity`でトグルして、フリップ中に2つの面が自然に切り替わる。

**レイアウトジャンプ防止：**

条件付き要素を`if`で追加/削除するとレイアウトがジャンプする。代わりに常にレンダリングし、`opacity`と`allowsHitTesting`で制御する：

```swift
AnswerButtonRow(intervals: intervals, onSelect: onGrade)
    .opacity(isFlipped ? 1 : 0)
    .allowsHitTesting(isFlipped)
```

**プレスフィードバック：**

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

## 多言語対応

### ランタイム言語切り替え

システム言語とは別に、アプリ内で言語を変更できる。`LanguageManager`が`@Published`で現在の`Bundle`を発行し、Presenterがこれを購読して言語変更時に自動的にViewStateを更新する。

```swift
// Presenterで言語変更を購読
LanguageManager.shared.$locale
    .dropFirst()
    .receive(on: RunLoop.main)
    .sink { [weak self] _ in self?.loadData() }
    .store(in: &cancellables)
```

String Catalog（`.xcstrings`）を使用してen、ko、ja、zh-Hansの4言語をサポートする。

---

## テスト戦略

Vladimir Khoриkovの**Unit Testing Principles, Practices, and Patterns**からインスピレーションを受けたテスト戦略を適用した。

### 核心原則

- **観察可能な動作**をテストし、内部実装はテストしない
- **ブラックボックステスト**：入力 → 出力のみ検証
- **Mockは共有依存性にのみ**：in-memoryの`ModelContainer`でDBを隔離

### テスト優先順位

1. **SRSEngine** — 最も重要なビジネスロジック。すべてのグレード組み合わせのinterval/EF計算を検証
2. **CSVParser** — 正常/異常CSV、BOM、クォートエスケープ
3. **Interactor** — CRUD動作（in-memory ModelContainer）
4. **Presenter** — ViewStateマッピング検証

```swift
// SRSEngineテスト例
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
    #expect(result.interval < 0.001)  // 約1分
}
```

SwiftDataテストではin-memoryの`ModelContainer`を使用してテスト間の隔離を保証する：

```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)
let container = try ModelContainer(for: Deck.self, Card.self, StudyLog.self,
                                    configurations: config)
let context = ModelContext(container)
let interactor = HomeInteractor(modelContext: context)
```

---

## 実戦で得たTips

### `navigationDestination(isPresented:)`は一つだけ

一つのViewに`navigationDestination(isPresented:)`を複数使うと、iOSナビゲーションバグが発生する。解決法はローカルenumで分岐すること：

```swift
enum DetailNavTarget {
    case flashCard(UUID)
    case editCard(UUID, UUID)
}

@State private var navTarget: DetailNavTarget?
@State private var showNav = false

// 一つのnavigationDestinationで複数の目的地を処理
.navigationDestination(isPresented: $showNav) {
    switch navTarget {
    case .flashCard(let id): FlashCardView(deckID: id)
    case .editCard(let deckID, let cardID): AddCardView(deckID: deckID, editingCardID: cardID)
    case nil: EmptyView()
    }
}
```

### 編集モードの再利用

「追加」と「編集」はUIがほぼ同じだ。別画面を作る代わりに、一つのViewを再利用する：

```swift
struct AddDeckView: View {
    let editingDeckID: UUID?  // nil = 作成、UUID = 編集

    init(editingDeckID: UUID? = nil) {
        self.editingDeckID = editingDeckID
    }
}
```

Presenterで`isEditMode`を判断して、ヘッダータイトルとボタンテキストを動的に変更する。

### Debounced Search with Combine

検索語が変わるたびにクエリを実行すると性能問題が生じる。Combineの`debounce`で200msの遅延を入れる：

```swift
$searchText
    .dropFirst()
    .debounce(for: .milliseconds(200), scheduler: RunLoop.main)
    .sink { [weak self] _ in self?.applySearch() }
    .store(in: &cancellables)
```

### ウィジェットデータ同期

WidgetKitはアプリとは別プロセスで実行される。App Groupsの`UserDefaults`を通じてデータを共有し、アプリがフォアグラウンドに来るたびにウィジェットタイムラインを更新する。

---

## 振り返り — 正直に

### 数字で見るプロジェクト

| 項目 | 数値 |
|------|------|
| Swiftファイル | 62個 |
| コード行数 | 約8,000行 |
| 画面（Scene） | 8個 |
| 再利用コンポーネント | 15個 |
| 対応言語 | 4言語（en、ko、ja、zh-Hans） |
| テストスイート | 6個 |
| 開発期間 | Phase 0〜5（UIプロトタイプ → ウィジェット/テスト） |

### うまくいったこと

**VIPをプロトコルなしで軽く適用したこと。** Bullet Journalではプロトコルベースのを使ったが、個人プロジェクトではオーバーヘッドだった。今回は実装の直接参照 + in-memory ModelContainerテストで簡潔に進め、テストにも問題なかった。

**SM-2を純粋関数enumとして隔離したこと。** アルゴリズムがアプリ全体で最も重要なビジネスロジックだが、副作用なしで入力 → 出力のみテストすればいい構造にしたことで、安心して修正できた。

**CSVインポートのユーザー体験。** 自動デリミタ検出、ヘッダースキップ、プレビューまで入れたら、「ファイル選択 → 確認」の2タップで完了する。元々作りたかったまさにそのフローだ。

### 残念だったこと

**CloudKitの制約を遅く把握したこと。** `unique`制約が使えないことをモデル設計後に知った。初期にPoCでCloudKit同期を先にテストしていれば、モデル構造が変わっていたかもしれない。新しいフレームワークを使う時はコア機能から検証するという教訓をまた一つ得た。

**テストカバレッジがSRSEngineとCSVParserに偏っていること。** InteractorとPresenterのテストも構造的には書けるが、機能実装に追われて後回しにした。「テストしやすい構造を作っておいてテストを書かない」というアイロニーはBullet Journalの時と同じだ。

**多言語文字列の検証。** 4言語をサポートしているが、中国語はネイティブの検証ができていない。機能的には動作するが、自然な表現かどうかは確認が必要だ。

---

## まとめ

このアプリを作って最も大きく学んだのは、**「制約が設計を決定する」**ということだ。

CloudKitの`unique`非対応がモデル設計を変え、SwiftUIの`@Environment`アクセス制限がDeferred Configureパターンを生み、iOSの`navigationDestination`バグがenumベースのルーティングを強制した。最初から予想した設計は一つもなかった。すべて制約にぶつかりながら出てきた結果物だ。

完璧なアプリではない。テストももっと書かなければならないし、多言語の検証も残っている。しかし「NotebookLMからCSVを出して、ワンクリックでインポートして、SM-2で復習する」という元々作りたかったフローは完成した。似たようなアプリを計画しているなら、この記事が回り道を減らす助けになれば嬉しい。
