# 5.5 Compose Multiplatform (UI 공유)

> Compose Multiplatform으로 로직뿐 아니라 **UI까지** iOS/Android에서 공유하는 방법을 학습한다.

---

## 학습 목표

- Compose Multiplatform(Compose MP)의 개념과 장/단점을 이해한다.
- 공용 Compose UI 코드가 두 플랫폼에서 동작함을 확인한다.
- 플랫폼별 뷰 통합(Android에선 Activity, iOS에선 UIViewController)을 한다.
- Material3, 네비게이션, 상태 관리까지 공용으로 구성한다.

---

## 1. Compose Multiplatform이란?

Jetpack Compose(Android UI 툴킷)를 iOS로 확장한 JetBrains 기술입니다.
같은 Composable 함수를 iOS와 Android 양쪽에서 렌더링합니다.

```
commonMain (Compose UI 코드)
    ├── Android → 컴파일 → Jetpack Compose(Android)
    └── iOS     → 컴파일 → Skia 기반 렌더러 (iOS)
```

**장점**: UI 개발 비용 대폭 절감, 로직+UI 완전 공용
**단점**: 네이티브 전용 UX(플랫폼 요소) 활용이 어려움, 퍼포먼스 미세조정 제약

> "로직만 공유"(5.1~5.4) vs "UI까지 공유"(이 챕터) 중 **프로젝트 목적과 팀 경험**으로 선택합니다.
> 대부분의 스타트업 앱 템플릿은 UI 공유를 기본으로 채택하고 있습니다.

---

## 2. 프로젝트 템플릿

JetBrains 공식 KMP 템플릿은 이미 Compose Multiplatform 옵션이 내장되어 있습니다.
- https://kmp.jetbrains.com/ 에서 "UI Preview(Compose Multiplatform)" 포함 선택

### 설정 (shared/build.gradle.kts)

```kotlin
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose")
    id("com.android.library")
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.ui)
            implementation(compose.components.resources)
            implementation("org.jetbrains.compose.navigation:navigation-compose:2.8.0-alpha")
        }
    }
}
```

- `compose.material3`가 기본 UI 라이브러리
- `compose.components.resources`로 리소스(이미지/문자열) 공용 제공

---

## 3. 공용 App composable 만들기

`commonMain`에 앱 루트 UI를 작성합니다.

```kotlin
// commonMain/kotlin/App.kt
import androidx.compose.runtime.*
import androidx.compose.material3.*

@Composable
fun App() {
    MaterialTheme(colorScheme = lightColorScheme()) {
        CounterScreen()
    }
}
```

```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Compose Multiplatform", fontSize = 24.sp)
        Text("카운트: $count", fontSize = 40.sp)

        Row {
            Button(onClick = { count++ }) { Text("+") }
            Button(onClick = { if (count > 0) count-- }) { Text("-") }
        }
    }
}
```

Windows/iOS 구분 없이 동일 코드입니다!

---

## 4. Android에서 실행

```kotlin
// androidApp/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            App()
        }
    }
}
```

- `App()` composable을 그대로 호출

---

## 5. iOS에서 실행 (UIViewController)

iOS는 Compose가 UIViewController 형태로 렌더링합니다.

```kotlin
// iosMain/kotlin/MainViewController.kt
import androidx.compose.ui.window.ComposeUIViewController

fun MainViewController() = ComposeUIViewController { App() }
```

**iOS 네이티브에서 보이기** — iosApp의 Swift 파일:

```swift
import SwiftUI
import UIKit
import Shared

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()   // Kotlin 함수 노출
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
    }
}

struct ContentView: View {
    var body: some View {
        ComposeView().ignoresSafeArea(.keyboard)
    }
}

#Preview {
    ContentView()
}
```

- `UIViewControllerRepresentable` = UIKit를 SwiftUI에서 임베드하는 래퍼
- 이제 iOS 시뮬레이터에서도 동일한 Counter 화면이 표시됩니다

---

## 6. 플랫폼 통합 지점 분리 (관심사 분리)

UI는 공용으로, 진입점/앱 수명주기는 플랫폼마다 다르게 둡니다.

```
commonMain  App(), 화면들, 상태, Resources
androidMain MainActivity (setContent → App)
iosMain     MainViewController (ComposeUIViewController → App)
```

| 책임 | 위치 |
|------|------|
| 화면/레이아웃/상태 | commonMain |
| Android 호스트 | androidMain(MainActivity) |
| iOS 호스트 | iosMain(MainViewController) |
| 푸시/스토어/딥링크 등 | 플랫폼 미들웨어(밖에서 연결) |

---

## 7. 네비게이션 (navigation-compose)

공용으로 화면 전환을 다룹니다.

```kotlin
import org.jetbrains.compose.navigation.NavController
import org.jetbrains.compose.navigation.composable
import org.jetbrains.compose.navigation.NavHost
import org.jetbrains.compose.navigation.rememberNavController

@Composable
fun RootNav() {
    val nav = rememberNavController()

    NavHost(navController = nav, startDestination = "home") {
        composable("home") {
            HomeScreen(
                onOpenDetail = { id -> nav.navigate("detail/$id") }
            )
        }
        composable("detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailScreen(id = id, onBack = { nav.popBackStack() })
        }
    }
}
```

---

## 8. 플랫폼별 컴포넌트가 필요할 때

간단 Alert 미세 조정이 필요하다면 다시 expect/actual:

```kotlin
// commonMain
expect fun toast(message: String)

// androidMain
actual fun toast(message: String) {
    // Toast.makeText(...)
}

// iosMain
actual fun toast(message: String) {
    // UIAlertController 또는 Swift 패스
}
```

---

## 9. 자원 공유: compose resources

`composeResources/` 폴더에 문자열/이미지 넣기:

```
shared/src/commonMain/composeResources/
├── drawable/my_image.xml
└── values/strings.xml
```

Kotlin에서:

```kotlin
import org.jetbrains.compose.resources.stringResource

Text(stringResource(Res.string.app_name))
// 또는
PainterResource(Res.drawable.my_image)
```

---

## 🛠 실습 1: 공용 "아이템 목록" 앱 UI

**요구사항**

1. commonMain에 `TodoComposeApp`:
   - 상단 제목, TextField + 추가 버튼
   - `LazyColumn` 리스트, 체크 토글, 삭제
2. 상태는 `remember { mutableStateListOf<TodoItem>() }` 사용
3. 공용 `data class TodoItem(id, title, done)` 정의
4. Android/iOS 양쪽에서 동일 화면 확인

**실습 힌트**

```kotlin
var todos by remember { mutableStateOf(listOf<TodoItem>()) }
```

체크 토글:

```kotlin
todos = todos.map { if (it.id == id) it.copy(done = !it.done) else it }
```

---

## 🛠 실습 2: 두 화면 전환 (네비게이션)

**요구사항**

1. HomeScreen(목록) + DetailScreen(선택 아이템 상세)
2. navigation-compose 사용
3. 문자열 파라미터 이동(`{id}`), 뒤로가기(`popBackStack`)
4. iOS/Android 각각 뒤로가기 제스처 동작 확인

---

## 🛠 실습 3: 그려내기(PainterRaw) — 온도 게이지 화면

**요구사항**

1. 공용 데이터: `val temps = listOf(24.5, 24.8, 25.1, 25.3, 25.6)`
2. Canvas로 막대 그래프를 직접 그리기 (2.6의 Combine과 대비)
3. Compose Canvas API 사용: `Canvas(modifier) { drawRect(...) }`
4. 각 막대 위에 온도 텍스트 표시

**실습 힌트**

```kotlin
Canvas(modifier = Modifier.fillMaxWidth().height(200.dp)) {
    val gap = size.width / temps.size
    temps.forEachIndexed { index, temp ->
        val h = (temp - 24.0) / 2.0 * size.height
        drawRect(
            color = Color(0xFF4FC3F7),
            topLeft = Offset(index * gap + 10f, size.height - h),
            size = Size(gap - 20f, h)
        )
    }
}
```

---

## 마무리 & 다음 챕터

Compose Multiplatform으로 UI까지 공용화했습니다. 이제 5단계(KMP) 완료입니다!

새 얼굴: `06_고급/01_아키텍처_패턴.md`로 아키텍처 설계를 학습합니다.