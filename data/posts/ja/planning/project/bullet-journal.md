---
title: Bullet Journal - 個人アプリにClean Architectureを適用して7画面を作った振り返り
date: 2026-02-22 13:00
excerpt: SwiftUI + SwiftData + VIP Architectureで集中タイマーアプリを作りながら経験したアーキテクチャの決定、技術的な壁、そして振り返り。
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

## 作ろうと思った理由

集中力アプリは世の中に溢れている。ポモドーロタイマー、フォーカスモード、森を育てるアプリ。全部試した。

問題はひとつだ。**タイマーをつけたままYouTubeを開いてしまう。** 意志力の問題というよりも、既存のアプリは「集中してください」と言うだけで、実際に集中を助ける環境を作ってくれないと感じていた。

自分が求めていたのはこういうものだった：

- タイマーを起動すると**雨の音が流れ**、ロック画面からもタイマーを操作できるアプリ
- 1日のスケジュールを**タイムラインで可視化**して、今何をすべきかすぐ分かるアプリ
- 1週間でどれだけ集中したか**統計で振り返れる**アプリ

ちょうどいいものがなかったので、自分で作ることにした。

<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ja/planning/project/bullet-journal-home_jp.jpg" alt="ホーム画面" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ja/planning/project/bullet-journal-timeline_jp.jpg" alt="タイムライン" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ja/planning/project/bullet-journal-dashboard_jp.jpg" alt="ダッシュボード" style="width: 32%; border-radius: 8px;" />
</div>

---

## 何を作ったのか

**Bullet Journal**は集中タイマー + スケジュール管理 + 統計ダッシュボードを組み合わせたiOSアプリだ。

| 画面 | 役割 |
|------|------|
| **ホーム** | 現在の時間帯のタスク表示 + 集中タイマー開始 |
| **集中モード** | 全画面タイマー + アンビエントサウンド + 一時停止/停止 |
| **タイムライン** | 1日のスケジュールを時間ブロックで可視化、タスクの作成/編集/削除 |
| **ダッシュボード** | 累計集中時間 + 週間チャート + 日別記録 |
| **デイリー記録** | 目標達成率、睡眠の質、今日の気分、振り返りの記入 |
| **設定** | 言語変更（韓/英/日）、アプリバージョン、プライバシーポリシー |
| **オンボーディング** | 初回起動時の紹介画面 |

さらに**ロック画面ウィジェット**（Small、Medium、LockScreen）と**ロック画面メディアコントロール**にも対応している。

技術スタック：

| 領域 | 選択 |
|------|------|
| UI | SwiftUI (iOS 17+) |
| データ | SwiftData |
| リアクティブ | Combine |
| アーキテクチャ | Clean Architecture (VIP) |
| 並行処理 | Swift 6 Strict Concurrency |
| 広告 | Google AdMob |
| 言語対応 | 韓国語、英語、日本語 |

---

## アーキテクチャ — なぜMVVMではなくVIPを選んだのか

最初はMVVMで始めようと思った。SwiftUIで最も自然なパターンだから。

しかしホーム画面ひとつに入るロジックを列挙してみたら、考えが変わった：

- SwiftDataから現在の時間帯のTask取得
- タイマーの開始/一時停止/再開/停止
- 前回セッションの累計時間計算 + 新規セッション保存
- アンビエントサウンドの再生/停止
- ロック画面メディアコントロールとの連携
- 睡眠の質の入力プロンプト
- ウィジェットタイムラインの更新

これをViewModel一つに入れたら500行のGod Objectになる。テストも諦めなければならない。

VIPはこの問題を構造的に分割してくれる：

<!-- VIPアーキテクチャダイアグラム挿入位置 -->
`[ダイアグラム挿入位置]`

```
ユーザーアクション → View → Interactor (ビジネスロジック)
                        ↓ Combine Publisher
                     Presenter (ViewModel変換)
                        ↓ @Published
                     View (画面更新)
```

- **View**はUIだけを描画する。ロジックはゼロ。
- **Interactor**はビジネスロジックをすべて持つ。SwiftDataクエリ、タイマー制御、セッション管理。
- **Presenter**はドメインモデルを画面に表示するViewModelに変換する。

実際のコードを見ると、ViewはDependencyを受け取ってInteractorを組み立て、Presenterを`@StateObject`として保持する：

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

PresenterはInteractorのCombine Publisherを購読して`@Published`プロパティを更新する。ここで`receive(on: DispatchQueue.main)`は使わない。すべてのサービスが`@MainActor`なので、Publisherはすでにメインスレッドで発行されているからだ。最初は習慣的に入れていたが、不要なスレッドホップが生じることに気づいて削除した。

このパターンで7つの画面をすべて作った。**4画面目からは新しい画面を追加するのにほとんど迷いがなくなった。** Interactor、Presenter、Modelsファイルを作って同じパターンで埋めれば終わりだ。

---

## 最も苦労した3つのこと

### 1. バックグラウンドタイマー — Timerはバックグラウンドで停止する

集中タイマーアプリで最も基本的な要件が最も厄介だった。iOSでは`Timer`はアプリがバックグラウンドに移ると停止する。ユーザーが画面を消して30分後に戻ってきたときにタイマーが0分で止まっていたら、アプリの存在意義がない。

解決戦略は**Timerに依存しないこと**だ。タイマーの開始時刻（`startedAt`）を記憶しておき、フォアグラウンド復帰時に`現在時刻 - startedAt`で経過時間を再計算する：

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

最初は`appDidEnterBackground`コールバック内にインラインでロジックを入れていたが、Race Conditionが発生した。NotificationCenterのコールバックとTimerのコールバックが同時に実行されて`elapsedSeconds`がおかしくなったのだ。ハンドラーを別メソッドに分離したら解決し、おまけにユニットテストも書けるようになった。

些細だが体感効果の大きいディテールがひとつ。Resume時に**即座にtickを送信する**。これをしないとRESUMEボタンを押した後、最大1秒間画面がフリーズして見える。`tickSubject.send(elapsedSeconds)` の1行がUXを大きく変えてくれる。

### 2. SwiftData — Enum Predicateがクラッシュする

SwiftDataを選んだのはCore DataよりSwift-nativeでコードがきれいだったからだ。ほとんどは満足していたが、ひとつ困惑する問題があった。

**EnumをPredicateで比較するとランタイムクラッシュが起きる：**

```swift
// クラッシュ
#Predicate<FocusSession> { $0.status == .completed }

// 変数に入れてもクラッシュ
let status = FocusSessionStatus.completed
#Predicate<FocusSession> { $0.status == status }
```

結局、解決法はシンプルだ。**全件取得してメモリ上でフィルタリングする：**

```swift
let all = try modelContext.fetch(FetchDescriptor<FocusSession>())
let completed = all.filter { $0.status == .completed }
```

個人アプリなのでデータが数千件を超えることはなく、パフォーマンスの問題はなかった。むしろ一度fetchして複数箇所で再利用するSingle Fetchパターンの方がコードがきれいになった。

教訓：**新しいフレームワークを使うときは、核となるクエリパターンをPoCで先に検証しよう。**

### 3. アンビエントサウンド — クロスフェードが予想以上に複雑

集中モードで雨の音や鳥の声などのアンビエントサウンドをループ再生する機能を入れた。再生自体は`AVAudioPlayer`で簡単だが、問題は**サウンドの切り替え**だった。

ユーザーが「雨の音」から「鳥の声」に変えるとき、音がプツッと途切れると不快だ。自然なクロスフェードが必要だった。前のサウンドを1秒間フェードアウトしながら新しいサウンドをフェードインするロジックを実装したが、0.05秒間隔でボリュームを調整するタイマーを別途管理する必要があった。

さらにロック画面メディアコントロール（`NowPlayingService`）との連携で複雑さが増した。ロック画面で一時停止を押すとタイマーも止まり、サウンドも止まらなければならない。この3つのサービス（Timer、AmbientSound、NowPlaying）の状態を`ServiceContainer`でまとめ、Interactorがオーケストレーションする構造で解決した：

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

    // テスト: Mock注入可能
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

プロダクションでは`ServiceContainer.shared`を使い、テストではMockを注入する。シングルトンだがDIが開いているのでテストに問題はない。

---

## 実践で得たTips 3つ

### Swift 6の@MainActorは本当に厳格

こんなコードがコンパイルエラーになる：

```swift
init(localizationManager: LocalizationManager = .shared) { }
```

`LocalizationManager.shared`が`@MainActor`なのに、default parameterは呼び出し時に評価されるためMainActorコンテキストが保証されないということだ。最初は「なぜこれがダメなの？」と戸惑ったが、結果的に明示的なDIを強制することになり、むしろテストしやすいコードになった。

### ウィジェットとアプリ間のデータ共有

ウィジェットはアプリとは別プロセスで実行される。同じSwiftDataを見るには**App Group**で共有コンテナを設定する必要がある。`AppConfiguration.sharedStoreURL`でパスを統一し、アプリでセッションが開始/終了するたびに`WidgetCenter.shared.reloadAllTimelines()`を呼び出してウィジェットを更新する。設定自体は簡単だが、これを見落とすとウィジェットが常に空の画面を表示するため、初期段階で対処すべき部分だ。

### 多言語対応は最初からやらないと後で苦しむ

3言語（韓国語、英語、日本語）に対応しているが、最初から`.xcstrings`で管理していたおかげで大きな困難はなかった。もし開発後半に入れようとしていたら、何百ものハードコードされた文字列をすべて探して置き換えなければならなかっただろう。

ひとつ落とし穴がある。SwiftUI Alertのtitleに文字列リテラルを入れるとローカライゼーションが効かない。必ず`Text()`で囲む必要がある。これを発見するのにかなり時間がかかった。

---

## 振り返り — 正直に

### 数字で見るプロジェクト

| 項目 | 数値 |
|------|------|
| Swiftファイル数 | 84個（アプリ68 + テスト7 + ウィジェット等） |
| 画面数 | 7個（すべてVIPパターン） |
| サービス | 3個（Timer、AmbientSound、NowPlaying） |
| ウィジェット | 4種（Small、Medium、LockScreen Circular、LockScreen Rectangular） |
| 対応言語 | 3言語（韓国語、英語、日本語） |

### うまくいったこと

**VIPを最後まで貫いたこと。** 7画面すべて同じ構造だ。新しい画面を追加するとき迷わず同じテンプレートに従えばいい。「このロジックはどこにある？」→ 常にInteractorだ。「表示形式を変えたい」→ 常にPresenterだ。探すのに5秒あれば十分だ。

**サービスをプロトコルで分離したこと。** TimerService、AmbientSoundService、NowPlayingServiceすべてにプロトコルがある。おかげでMockを作ってInteractorを独立してテストできた。

**バックグラウンドタイマーをきちんと処理したこと。** ハンドラーを別メソッドに分離し、時間差計算方式を使ったおかげで、画面ロック後に復帰してもタイマーが正確だ。

### 残念だったこと

**テストを十分に書けなかったこと。** VIP構造のおかげでテストが書きやすい構造なのに、機能実装に追われてカバレッジが低い。「テストしやすい構造を作っておいてテストを書かなかった」というアイロニーになってしまった。

**SwiftDataを信頼しすぎたこと。** Appleのファーストパーティフレームワークだから当然うまくいくだろうと思っていた。Enum Predicateのクラッシュを初期段階で知っていたら、モデル設計が変わっていたかもしれない。

### 個人プロジェクトにClean Architectureは過剰か？

結論から言えば**そうではない。** 一人で作るアプリだから過剰に感じるかもしれないが、2週間後にコードを開き直したときに真価を発揮する。

ただし画面が2〜3個のシンプルなアプリならMVVMで十分だ。VIPは画面が多くビジネスロジックが複雑なときに力を発揮する。このアプリは7画面にタイマー、サウンド、ロック画面、ウィジェットまで絡んでいるのでVIPが正しい選択だった。

---

## おわりに

このアプリを作りながら最も大きく学んだのは、**「構造こそが速度」** ということだ。

序盤にアーキテクチャを決める時間がもったいなく感じる。しかし7番目の画面を作るとき、バックグラウンドタイマーのバグを直すとき、ウィジェットから同じデータを読む必要があるとき――その投資が何倍にもなって返ってくる。

完璧なアプリではない。まだやりたいことはたくさんある。しかし作る過程でSwiftUI、SwiftData、Combine、Swift Concurrencyを実践レベルで経験できた。同じようなアプリを計画している人がいれば、この記事が試行錯誤を減らす助けになれば幸いだ。
