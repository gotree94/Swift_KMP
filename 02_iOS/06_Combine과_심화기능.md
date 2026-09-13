# 2.6 Combine과 심화 기능

> Combine 프레임워크로 이벤트 스트림을 다루고, 실전 iOS 앱을 완성도 있게 만드는 고급 기능을 학습한다.

---

## 학습 목표

- Combine의 개념(Publisher/Subscriber)을 이해한다.
- `@Published`, `CurrentValueSubject`, `PassthroughSubject`를 사용할 수 있다.
- 애니메이션, 이미지 캐시, 타이머 등 실제 앱에 유용한 기능을 익힌다.
- SwiftUI + URLSession + Combine을 결합한 실전 코드를 작성한다.

---

## 1. Combine이란?

**시간에 따라 변하는 값의 스트림**을 다루는 프레임워크입니다.

- **Publisher**: 값이 나오는 쪽 (스트림)
- **Subscriber**: 값을 구독해 소비하는 쪽
- **Operator**: map/filter/first 등 스트림 변환

```swift
import Combine

let publisher = [1, 2, 3].publisher

publisher
    .map { $0 * 10 }
    .sink { value in
        print(value)            // 10, 20, 30
    }
```

---

## 2. @Published와 ObservableObject

UIKit(Storyboard) 프로젝트에서는 `@Published` 기반 Combine이 많이 쓰였습니다. 지금도 이해하면 큰 도움이 됩니다.

```swift
import Combine

class SettingsViewModel: ObservableObject {
    @Published var userName = "게스트"
    @Published var messageCount = 0

    func login() {
        userName = "Minho"
        messageCount = 5
    }
}
```

View에 연결:

```swift
struct SettingsScreen: View {
    @StateObject private var viewModel = SettingsViewModel()

    var body: some View {
        VStack {
            Text("\(viewModel.userName)님, 메시지 \(viewModel.messageCount)개")
            Button("로그인") { viewModel.login() }
        }
    }
}
```

> `@Published` 값이 바뀔 때마다 뷰가 자동 갱신됩니다. @Observable과 비슷하지만 Combine 전통 방식입니다.

---

## 3. Subject: 스트림의 중계

Combine에서 Publisher에 값을 직접 넣으려면 `Subject`를 사용합니다.

```swift
import Combine

let subject = PassthroughSubject<String, Never>()   // 값만 전달
let curValue = CurrentValueSubject<Int, Never>(0)   // 마지막 값 기억

let cancel = subject.sink { value in
    print("받음: \(value)")
}

subject.send("첫 메시지")   // 받음: 첫 메시지
subject.send("둘째")        // 받음: 둘째

cancel.cancel()             // 구독 종료 (메모리 누수 방지)
```

---

## 4. Timer 예제 (워치 앱)

```swift
import SwiftUI
import Combine

struct StopwatchView: View {
    @State private var seconds = 0
    @State private var timer: AnyCancellable?

    var body: some View {
        VStack(spacing: 30) {
            Text(timeString(seconds))
                .font(.system(size: 64, weight: .bold, design: .monospaced))
            HStack {
                Button(timer == nil ? "시작" : "중지") {
                    toggle()
                }
                .buttonStyle(.borderedProminent)
                Button("리셋") { seconds = 0 }
                    .buttonStyle(.bordered)
                    .disabled(timer != nil)
            }
        }
        .onDisappear { timer?.cancel() }   // 화면 이탈 시 정리
    }

    func timeString(_ s: Int) -> String {
        String(format: "%02d:%02d", s / 60, s % 60)
    }

    func toggle() {
        if timer == nil {
            timer = Timer.publish(every: 1, on: .main, in: .common)
                .autoconnect()
                .sink { _ in seconds += 1 }
        } else {
            timer?.cancel()
            timer = nil
        }
    }
}
```

---

## 5. 이미지 로딩 + 캐시

리스트의 이미지를 매번 네트워크로 받지 않도록 캐시해 봅니다.

```swift
import SwiftUI

actor ImageCache {
    static let shared = ImageCache()
    private var store: [String: Data] = [:]

    func data(for key: String) -> Data? {
        store[key]
    }

    func set(_ data: Data, for key: String) {
        store[key] = data
    }
}

struct CachedAsyncImage: View {
    let url: URL?

    @State private var image: UIImage?

    var body: some View {
        Group {
            if let image {
                Image(uiImage: image)
                    .resizable()
                    .scaledToFit()
            } else {
                ProgressView()
            }
        }
        .task { await load() }
    }

    func load() async {
        guard let url, image == nil else { return }

        // 캐시 먼저 확인
        if let cached = await ImageCache.shared.data(for: url.absoluteString),
           let loaded = UIImage(data: cached) {
            image = loaded
            return
        }

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            await ImageCache.shared.set(data, for: url.absoluteString)
            image = UIImage(data: data)
        } catch {
            print("이미지 로드 실패: \(error)")
        }
    }
}
```

---

## 6. 애니메이션

### withAnimation

```swift
struct AnimationDemo: View {
    @State private var expanded = false

    var body: some View {
        VStack {
            RoundedRectangle(cornerRadius: expanded ? 24 : 8)
                .fill(expanded ? Color.blue : Color.orange)
                .frame(width: expanded ? 300 : 120,
                       height: expanded ? 200 : 80)
                .rotationEffect(.degrees(expanded ? 45 : 0))

            Button("토글") {
                withAnimation(.spring(duration: 0.4)) {
                    expanded.toggle()
                }
            }
        }
    }
}
```

### SymbolEffect & canvas (iOS 17+)

```swift
Image(systemName: "heart.fill")
    .symbolEffect(.bounce, value: isLiked)   // 아이콘 바운스
```

### 커스텀 트랜지션

```swift
if showDetail {
    DetailView()
        .transition(.move(edge: .bottom).combined(with: .opacity))
}
```

---

## 7. 실전 예제: 포커스 타이머 앱 (종합)

이제 지금까지 배운 것들을 조합한 앱을 만듭니다.

**요구사항**

1. `Timer` 기반 25분 포커스 타이머
2. 시작/중지/리셋, 완료 시 toast(경고)
3. 세션 완료 횟수를 UserDefaults 저장
4. ProgressView로 진행률 표시

```swift
import SwiftUI

struct FocusTimerView: View {
    @State private var remaining = 25 * 60
    @State private var isRunning = false
    @State private var timerTask: Task<Void, Never>?

    @State private var completedSessions =
        UserDefaults.standard.integer(forKey: "focusSessions")

    var body: some View {
        VStack(spacing: 30) {
            Text("남은 시간")
                .font(.headline)
            Text(String(format: "%02d:%02d", remaining / 60, remaining % 60))
                .font(.system(size: 72, weight: .bold, design: .monospaced))
            ProgressView(value: Double(remaining), total: 1500)

            HStack(spacing: 20) {
                Button(isRunning ? "일시정지" : "시작") {
                    toggle()
                }
                .buttonStyle(.borderedProminent)
                Button("리셋") {
                    remaining = 1500
                }
                .buttonStyle(.bordered)
                .disabled(remaining == 1500)
            }

            Text("완료한 세션: \(completedSessions)회")
                .foregroundStyle(.secondary)
        }
        .padding()
        .onDisappear { timerTask?.cancel() }
    }

    func toggle() {
        if isRunning {
            timerTask?.cancel()
        } else {
            timerTask = Task {
                while remaining > 0 {
                    try? await Task.sleep(for: .seconds(1))
                    remaining -= 1
                }
                if remaining == 0 {
                    completedSessions += 1
                    UserDefaults.standard.set(completedSessions, forKey: "focusSessions")
                    // 완료 처리 (경고/로컬 알림 등)
                }
            }
        }
        isRunning.toggle()
    }
}
```

---

## 🛠 실습: 정지 화면(스톱워치) + 애니메이션 결합 앱

**요구사항**

1. 시작/중지/리셋 스톱워치 (Combine Timer)
2. 시작 중일 때 원형 ProgressRing이 회전하는 애니메이션
3. 달성한 랩타임 기록 List 표시 & UserDefaults 저장
4. 10초마다 자동으로 랩 기록(자동 기록) 추가

**실습 힌트**

- 원형 표시: `Circle().trim(from:0, to: progress).stroke(style: StrokeStyle(lineWidth: 10, lineCap: .round))` + `.rotationEffect(-90°)`
- 숫자 업데이트: `Timer.publish(...).sink { }`

---

## 마무리 & 다음 챕터

Combine과 심화 UI 기능으로 iOS 앱의 완성도를 높였습니다.

이제 iOS 파트가 끝났습니다! 다음은 **그린 필드**: `03_Kotlin/01_기초_문법.md` 로 이동해 Kotlin 언어를 학습합니다.