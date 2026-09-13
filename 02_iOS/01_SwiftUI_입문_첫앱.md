# 2.1 SwiftUI 입문: 첫 iOS 앱 만들기

> Xcode에서 첫 SwiftUI 앱을 만들고, View의 구조와 기본 구성 요소를 학습한다.

---

## 학습 목표

- Xcode 프로젝트를 만들고 시뮬레이터에서 실행할 수 있다.
- SwiftUI `View`의 동작 방식(@main, body)을 이해한다.
- Text, Image, Button 등 기본 컴포넌트를 사용할 수 있다.
- Preview를 활용해 UI를 빠르게 확인할 수 있다.

---

## 1. 첫 프로젝트 만들기

### 절차

1. Xcode 실행 → `Create a New Project...`
2. iOS 탭에서 **App** 선택 → Next
3. 옵션 설정:
   - Product Name: `MyFirstApp`
   - Interface: **SwiftUI**
   - Language: **Swift**
   - (필요 시 Team: 본인 Apple ID)
4. 저장 위치 선택 → Create
5. `⌘ + R` (빌드 & 실행) → iPhone 시뮬레이터에서 앱 확인

> 📌 **주의**: `Interface`가 Storyboard(UIKit)가 아니고 **SwiftUI**인지 확인하세요.

---

## 2. 프로젝트 구조 이해

생성된 `ContentView.swift`를 열어봅니다.

```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        VStack {
            Image(systemName: "globe")
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text("Hello, world!")
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

구성 요소:

| 요소 | 의미 |
|------|------|
| `struct ContentView: View` | 화면 하나를 나타내는 View 구조체 |
| `var body: some View` | 뷰의 내용을 그리는 필수 프로퍼티 |
| `VStack` / `HStack` | 세로 / 가로 배치 |
| `Image(systemName:)` | SF Symbol(아이콘) 표시 |
| `Text("...")` | 텍스트 표시 |
| `#Preview` | 캔버스 미리보기 |

`MyFirstAppApp.swift` (앱의 진입점):

```swift
import SwiftUI

@main
struct MyFirstAppApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

- `@main` + `App` 프로토콜 → 앱 실행 시 `body`의 뷰를 표시합니다.

---

## 3. View의 기본 개념

### View = 구조체 + 프로퍼티

화면은 항상 **데이터(프로퍼티)에 따라 재계산**됩니다. SwiftUI는 상태가 바뀌면 해당 부분만 다시 그립니다.

```swift
struct GreetingView: View {
    let name: String       // 입력 데이터

    var body: some View {
        Text("안녕, \(name)!")
            .font(.largeTitle)   // 수정자(modifier) 체이닝
            .foregroundStyle(.blue)
            .padding()
            .background(Color.yellow.opacity(0.3))
            .cornerRadius(12)
    }
}
```

### 수정자 (Modifier)

뷰에 스타일/레이아웃을 적용하는 체이닝 함수들입니다.

```swift
Text("SwiftUI")
    .font(.system(size: 30, weight: .bold))
    .italic()
    .underline(true, color: .red)
    .padding()
    .frame(maxWidth: .infinity)          // 최대 폭 확장
    .background(LinearGradient(
        colors: [.green, .blue],
        startPoint: .leading,
        endPoint: .trailing
    ))
```

---

## 4. 기본 컴포넌트 익히기

### Text (텍스트)

```swift
Text("기본 텍스트")
    .font(.headline)
    .fontWeight(.semibold)
    .lineLimit(2)              // 최대 2줄
    .multilineTextAlignment(.center)
```

### Image (이미지)

```swift
// SF Symbol(시스템 아이콘)
Image(systemName: "star.fill")
    .font(.system(size: 60))
    .foregroundStyle(.yellow)

// 색 커스텀
Image(systemName: "heart")
    .foregroundStyle(.red)
    .scaleEffect(2.0)          // 크기 배율
```

### Button (버튼)

```swift
Button("눌러보세요") {
    print("버튼이 눌렸습니다!")
}
.buttonStyle(.borderedProminent)   // 강조 버튼 스타일
```

버튼에 아이콘+텍스트 조합:

```swift
Button(action: {
    print("좋아요!")
}) {
    Label("좋아요", systemImage: "heart")
        .font(.title2)
        .padding()
}
.buttonStyle(.borderedProminent)
.tint(.pink)
```

### TextField (입력)

```swift
@State private var input = ""

TextField("이름을 입력하세요", text: $input)
    .textFieldStyle(.roundedBorder)
    .padding()
```

> `@State`와 `$` 바인딩은 다음 챕터 `02_레이아웃과_데이터_흐름.md`에서 자세히 다룹니다.

### Spacer (여백)

```swift
HStack {
    Text("왼쪽")
    Spacer()        // 남는 공간을 밀어냄
    Text("오른쪽")
}
```

---

## 5. 화면 구성 실전 예제

`ContentView.swift`를 통째로 아래로 교체해 보세요.

```swift
import SwiftUI

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 20) {
            Text("카운터 앱")
                .font(.largeTitle)
                .fontWeight(.bold)

            Text("현재 값: \(count)")
                .font(.title)
                .monospacedDigit()

            HStack(spacing: 30) {
                Button("-") { if count > 0 { count -= 1 } }
                Button("+") { count += 1 }
            }
            .buttonStyle(.borderedProminent)
            .font(.title2)

            Spacer()

            if count >= 10 {
                Text("🐐 GOAT 상태!")
                    .font(.headline)
                    .foregroundStyle(.orange)
            }
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

캔버스에서 버튼을 눌러 카운트가 변하는지 확인하고, 시뮬레이터에서도 실행합니다.

---

## 🛠 실습: 나만의 명함(Business Card) 앱

신규 프로젝트 `MyCard`를 만들고 `ContentView.swift`를 바꿔봅니다.

**요구사항**

1. SF Symbol 사진 아이콘(`person.crop.circle`) + 이름 + 직책
2. `Divider()`로 구분선
3. 이메일, 전화, 블로그 정보를 아이콘 + 텍스트로 표시
4. 배경색을 그라데이션으로 꾸미기
5. 하단에 "GitHub 방문" 버튼 (눌리면 print)

```swift
// 참고: VStack/HStack/Spacer/Divider/Image/Text/Button 조합
```

### 실습 힌트

```swift
VStack(spacing: 16) {
    Image(systemName: "person.crop.circle.fill")
        .resizable()
        .frame(width: 120, height: 120)
        .foregroundStyle(.teal)

    Text("김민호")
        .font(.largeTitle).fontWeight(.bold)
    ...
}
```

- `.resizable()` + `.frame()` 는 아이콘 크기 조절에 꼭 필요합니다.
- HStack의 라벨: `Label("minho@example.com", systemImage: "envelope")`

---

## 마무리 & 다음 챕터

이제 SwiftUI의 기본 뷰 동작을 이해했습니다.

다음 챕터 `02_레이아웃과_데이터_흐름.md`에서 스택 레이아웃 정리와 `@State`/`@Binding`/`@Observable`에 의한 상태 관리(데이터 흐름)를 깊게 학습합니다.