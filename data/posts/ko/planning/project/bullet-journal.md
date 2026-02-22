---
title: Bullet Journal - 개인 앱에 Clean Architecture를 적용하고 7개 화면을 만든 후기
date: 2026-02-22 13:00
excerpt: SwiftUI + SwiftData + VIP Architecture로 집중 타이머 앱을 만들면서 겪은 아키텍처 결정, 기술적 난관, 그리고 회고.
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

## 만들게 된 이유

집중력 앱은 세상에 넘친다. 뽀모도로 타이머, 포커스 모드, 숲 키우기. 다 써봤다.

문제는 하나다. **타이머를 켜놓고 유튜브를 연다.** 의지력의 문제라기보다, 기존 앱들이 "집중하세요"라고 말해줄 뿐 실제로 집중을 도와주는 환경을 만들어주지 않는다고 느꼈다.

내가 원하는 건 이런 거였다:

- 타이머를 켜면 **빗소리가 깔리고**, 잠금화면에서도 타이머를 제어할 수 있는 앱
- 하루 일정을 **타임라인으로 시각화**해서, 지금 뭘 해야 하는지 바로 알 수 있는 앱
- 한 주간 얼마나 집중했는지 **통계로 돌아볼 수 있는** 앱

마땅한 게 없어서 직접 만들기로 했다.

<div style="display: flex; gap: 12px; margin: 1rem 0;">
  <img src="/images/posts/ko/planning/project/bullet-journal-home_ko.jpg" alt="홈 화면" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/bullet-journal-timeline_ko.jpg" alt="타임라인" style="width: 32%; border-radius: 8px;" />
  <img src="/images/posts/ko/planning/project/bullet-journal-dashboard_ko.jpg" alt="대시보드" style="width: 32%; border-radius: 8px;" />
</div>

---

## 무엇을 만들었나

**Bullet Journal**은 집중 타이머 + 일정 관리 + 통계 대시보드를 합친 iOS 앱이다.

| 화면 | 역할 |
|------|------|
| **홈** | 현재 시간대의 할 일 표시 + 집중 타이머 시작 |
| **집중 모드** | 전체화면 타이머 + 앰비언트 사운드 + 일시정지/정지 |
| **타임라인** | 하루 일정을 시간 블록으로 시각화, 할 일 생성/편집/삭제 |
| **대시보드** | 누적 집중 시간 + 주간 차트 + 일별 기록 |
| **일일 기록** | 목표 달성률, 수면 품질, 오늘의 기분, 회고 작성 |
| **설정** | 언어 변경(한/영/일), 앱 버전, 개인정보 처리방침 |
| **온보딩** | 첫 실행 시 소개 화면 |

여기에 **잠금화면 위젯**(Small, Medium, LockScreen)과 **잠금화면 미디어 컨트롤**도 지원한다.

기술 스택:

| 영역 | 선택 |
|------|------|
| UI | SwiftUI (iOS 17+) |
| 데이터 | SwiftData |
| 리액티브 | Combine |
| 아키텍처 | Clean Architecture (VIP) |
| 동시성 | Swift 6 Strict Concurrency |
| 광고 | Google AdMob |
| 언어 지원 | 한국어, 영어, 일본어 |

---

## 아키텍처 — 왜 MVVM 대신 VIP를 골랐나

처음엔 MVVM으로 시작하려 했다. SwiftUI에서 가장 자연스러운 패턴이니까.

그런데 홈 화면 하나에 들어가는 로직을 나열해보니 마음이 바뀌었다:

- SwiftData에서 현재 시간대 Task 조회
- 타이머 시작/일시정지/재개/정지
- 이전 세션 누적 시간 계산 + 새 세션 저장
- 앰비언트 사운드 재생/정지
- 잠금화면 미디어 컨트롤 연동
- 수면 품질 입력 프롬프트
- 위젯 타임라인 갱신

이걸 ViewModel 하나에 넣으면 500줄짜리 God Object가 된다. 테스트도 포기해야 한다.

VIP는 이 문제를 구조적으로 나눠준다:

<!-- VIP 아키텍처 다이어그램 삽입 위치 -->
`[다이어그램 삽입 위치]`

```
사용자 액션 → View → Interactor (비즈니스 로직)
                        ↓ Combine Publisher
                     Presenter (ViewModel 변환)
                        ↓ @Published
                     View (화면 갱신)
```

- **View**는 UI만 그린다. 로직 제로.
- **Interactor**는 비즈니스 로직을 전부 가진다. SwiftData 쿼리, 타이머 제어, 세션 관리.
- **Presenter**는 도메인 모델을 화면에 표시할 ViewModel으로 변환한다.

실제 코드를 보면, View는 의존성을 받아서 Interactor를 조립하고, Presenter를 `@StateObject`로 소유한다:

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

Presenter는 Interactor의 Combine Publisher를 구독해서 `@Published` 프로퍼티를 업데이트한다. 여기서 `receive(on: DispatchQueue.main)`을 쓰지 않는다. 모든 서비스가 `@MainActor`라서 Publisher가 이미 메인 스레드에서 발행되기 때문이다. 처음엔 습관적으로 넣었다가, 불필요한 스레드 홉이 생긴다는 걸 알고 제거했다.

이 패턴으로 7개 화면을 전부 만들었다. **4번째 화면부터는 새 화면을 추가하는 데 거의 고민이 없었다.** Interactor, Presenter, Models 파일을 만들고 같은 패턴으로 채우면 끝이다.

---

## 가장 어려웠던 3가지

### 1. 백그라운드 타이머 — Timer는 백그라운드에서 죽는다

집중 타이머 앱에서 가장 기본적인 요구사항이 가장 까다로웠다. iOS에서 `Timer`는 앱이 백그라운드에 가면 멈춘다. 사용자가 화면을 끄고 30분 후에 돌아왔는데 타이머가 0분에 멈춰 있으면 앱의 존재 이유가 없다.

해결 전략은 **Timer에 의존하지 않는 것**이다. 타이머 시작 시각(`startedAt`)을 기억해두고, 포그라운드 복귀 시 `현재 시각 - startedAt`으로 경과 시간을 재계산한다:

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

처음에는 `appDidEnterBackground` 콜백 안에 인라인으로 로직을 넣었다가 Race Condition이 발생했다. NotificationCenter 콜백과 Timer 콜백이 동시에 실행되면서 `elapsedSeconds`가 꼬인 것이다. 핸들러를 별도 메서드로 분리하니 해결됐고, 덤으로 유닛 테스트도 작성할 수 있게 됐다.

사소하지만 체감이 큰 디테일 하나. Resume 시 **즉시 tick을 전송**한다. 이걸 안 하면 RESUME 버튼을 누른 후 최대 1초간 화면이 멈춰 보인다. `tickSubject.send(elapsedSeconds)` 한 줄이 UX를 확 바꿔준다.

### 2. SwiftData — Enum Predicate가 크래시난다

SwiftData를 선택한 건 Core Data보다 Swift-native하고 코드가 깔끔해서였다. 대부분은 만족했는데, 하나 당황스러운 문제가 있었다.

**Enum을 Predicate에서 비교하면 런타임 크래시가 난다:**

```swift
// 크래시
#Predicate<FocusSession> { $0.status == .completed }

// 변수에 담아도 크래시
let status = FocusSessionStatus.completed
#Predicate<FocusSession> { $0.status == status }
```

결국 해결법은 단순하다. **전부 가져와서 메모리에서 필터링한다:**

```swift
let all = try modelContext.fetch(FetchDescriptor<FocusSession>())
let completed = all.filter { $0.status == .completed }
```

개인 앱이라 데이터가 수천 건을 넘지 않으므로 성능 문제는 없었다. 오히려 한 번 fetch해서 여러 곳에 재사용하는 Single Fetch 패턴이 코드를 더 깔끔하게 만들어줬다.

교훈: **새로운 프레임워크를 쓸 때는 핵심 쿼리 패턴을 PoC로 먼저 검증하자.**

### 3. 앰비언트 사운드 — 크로스페이드가 예상보다 복잡하다

집중 모드에서 빗소리, 새소리 같은 앰비언트 사운드를 루프 재생하는 기능을 넣었다. 재생 자체는 `AVAudioPlayer`로 간단하지만, 문제는 **사운드 전환**이었다.

사용자가 "빗소리"에서 "새소리"로 바꿀 때 소리가 뚝 끊기면 불쾌하다. 자연스러운 크로스페이드가 필요했다. 이전 사운드를 1초간 페이드아웃하면서 새 사운드를 페이드인하는 로직을 구현했는데, 0.05초 간격으로 볼륨을 조절하는 타이머를 별도로 관리해야 했다.

여기에 잠금화면 미디어 컨트롤(`NowPlayingService`)까지 연동하니 복잡도가 올라갔다. 잠금화면에서 일시정지를 누르면 타이머도 멈추고 사운드도 멈춰야 한다. 이 세 서비스(Timer, AmbientSound, NowPlaying)의 상태를 `ServiceContainer`로 묶고, Interactor가 오케스트레이션하는 구조로 풀었다:

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

프로덕션에서는 `ServiceContainer.shared`를 쓰고, 테스트에서는 Mock을 주입한다. 싱글톤이지만 DI가 열려 있어서 테스트에 문제가 없다.

---

## 실전에서 건진 팁 3가지

### Swift 6의 @MainActor는 진짜 엄격하다

이런 코드가 컴파일 에러를 낸다:

```swift
init(localizationManager: LocalizationManager = .shared) { }
```

`LocalizationManager.shared`가 `@MainActor`인데, default parameter는 호출 시점에 평가되므로 MainActor 컨텍스트가 보장되지 않는다는 것이다. 처음엔 "이게 왜 안 돼?"라고 당황했지만, 결과적으로 명시적 DI를 강제하게 되면서 오히려 테스트하기 좋은 코드가 됐다.

### 위젯과 앱 사이의 데이터 공유

위젯은 앱과 별도 프로세스에서 실행된다. 같은 SwiftData를 보려면 **App Group**으로 공유 컨테이너를 설정해야 한다. `AppConfiguration.sharedStoreURL`로 경로를 통일하고, 앱에서 세션이 시작/종료될 때마다 `WidgetCenter.shared.reloadAllTimelines()`를 호출해서 위젯을 갱신한다. 설정 자체는 간단하지만, 이걸 빠뜨리면 위젯이 항상 빈 화면을 보여주기 때문에 초반에 잡아야 하는 부분이다.

### 다국어는 처음부터 하지 않으면 나중에 고통받는다

3개 언어(한국어, 영어, 일본어)를 지원하는데, 처음부터 `.xcstrings`로 관리한 덕에 큰 어려움은 없었다. 만약 개발 후반에 넣으려 했다면 수백 개의 하드코딩된 문자열을 전부 찾아 바꿔야 했을 것이다.

한 가지 함정이 있다. SwiftUI Alert의 title은 문자열 리터럴을 넣으면 로컬라이제이션이 안 된다. 반드시 `Text()`로 감싸야 한다. 이걸 발견하는 데 꽤 오래 걸렸다.

---

## 회고 — 솔직하게

### 숫자로 보는 프로젝트

| 항목 | 수치 |
|------|------|
| Swift 파일 수 | 84개 (앱 68 + 테스트 7 + 위젯 등) |
| 화면 수 | 7개 (전부 VIP 패턴) |
| 서비스 | 3개 (Timer, AmbientSound, NowPlaying) |
| 위젯 | 4종 (Small, Medium, LockScreen Circular, LockScreen Rectangular) |
| 지원 언어 | 3개 (한국어, 영어, 일본어) |

### 잘한 것

**VIP를 끝까지 지킨 것.** 7개 화면 모두 동일한 구조다. 새 화면을 추가할 때 고민 없이 같은 템플릿을 따르면 된다. "이 로직이 어디 있지?" → 항상 Interactor다. "표시 형식을 바꾸고 싶다" → 항상 Presenter다. 찾는 데 5초면 된다.

**서비스를 프로토콜로 분리한 것.** TimerService, AmbientSoundService, NowPlayingService 모두 프로토콜이 있다. 덕분에 Mock을 만들어서 Interactor를 독립적으로 테스트할 수 있었다.

**백그라운드 타이머를 제대로 처리한 것.** 핸들러를 별도 메서드로 분리하고, 시간 차이 계산 방식을 쓴 덕에 화면 잠금 후 복귀해도 타이머가 정확하다.

### 아쉬운 것

**테스트를 충분히 못 쓴 것.** VIP 구조 덕분에 테스트 작성이 쉬운 구조인데, 기능 구현에 쫓겨 커버리지가 낮다. "테스트하기 좋은 구조를 만들어 놓고 테스트를 안 쓴" 아이러니가 됐다.

**SwiftData를 너무 믿은 것.** Apple의 1st-party 프레임워크니까 당연히 잘 되겠지, 라고 생각했다. Enum Predicate 크래시를 초반에 알았다면 모델 설계를 달리했을 수 있다.

### 개인 프로젝트에 Clean Architecture가 과한가?

결론부터 말하면 **아니다.** 혼자 만드는 앱이라 과하게 느껴질 수 있지만, 2주 뒤에 코드를 다시 열었을 때 진가가 발휘된다.

다만 화면이 2~3개인 간단한 앱이라면 MVVM으로 충분하다. VIP는 화면이 많고 비즈니스 로직이 복잡할 때 빛을 발한다. 이 앱은 7개 화면에 타이머, 사운드, 잠금화면, 위젯까지 엮여 있으니 VIP가 맞는 선택이었다.

---

## 마무리

이 앱을 만들면서 가장 크게 배운 건 **"구조가 곧 속도"** 라는 것이다.

초반에 아키텍처를 잡는 시간이 아깝게 느껴진다. 하지만 7번째 화면을 만들 때, 백그라운드 타이머 버그를 고칠 때, 위젯에서 같은 데이터를 읽어야 할 때 — 그 투자가 몇 배로 돌아온다.

완벽한 앱은 아니다. 아직 하고 싶은 게 많다. 하지만 만드는 과정에서 SwiftUI, SwiftData, Combine, Swift Concurrency를 실전 수준으로 경험할 수 있었다. 비슷한 앱을 계획하고 있다면, 이 글이 삽질을 줄이는 데 도움이 되면 좋겠다.
