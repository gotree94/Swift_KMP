# 5.1 KMP 개념과 프로젝트 구조

> Kotlin Multiplatform(KMP)이 무엇이고, 어떤 구조로 동작하는지 이해하고 첫 KMP 프로젝트를 생성한다.

---

## 학습 목표

- KMP가 지원하는 플랫폼과 동작 원리를 이해한다.
- 공용 모듈(shared)과 플랫폼 모듈(iOS/Android) 구조를 이해한다.
- KMP 프로젝트를 생성하고 빌드할 수 있다.
- 공용 코드가 어떻게 그 각 플랫폼에서 실행되는지 확인한다.

---

## 1. KMP란 무엇인가?

Kotlin Multiplatform은 **비즈니스 로직(네트워킹, 데이터 처리, 저장 등)을 한 번 작성해
iOS와 Android 양쪽에서 재사용**하는 JetBrains의 기술입니다.

```
          ┌─────────────────────────────┐
          │  Kotlin 공용 모듈 (shared)  │  ← 한 번 작성
          │  네트워킹·저장·로직·모델    │
          └──────────┬──────────────────┘
                     ▼
   ┌───────────────────────┐      ┌───────────────────────┐
   │      Android 앱        │      │        iOS 앱         │
   │  Jetpack Compose UI    │      │   SwiftUI/Native UI   │
   │  공용 코드 호출         │      │  공용 코드 호출(Kotlin →  │
   │  (직접, 바로 호출)      │      │   Swift/Framework)    │
   └───────────────────────┘      └───────────────────────┘
```

**핵심 장점**
- 비즈니스 로직을 한 번만 작성 → 중복 코드 2배 감소
- 로직을 공용으로 만들면 한 곳에서 고칠 수 있음
- iOS도 Kotlin으로 짠 로직을 Swift에서 호출

> 참고: KMP는 "UI까지 공유" 할 수도 있고(Compose Multiplatform, 5.5), 
> "로직만 공유" 할 수도 있습니다(네이티브 UI + shared 모듈). 여기서는 로직 공유를 먼저 익힙니다.

---

## 2. 프로젝트 구조

KMP는 3개 영역으로 나뉩니다.

```
mykmp/
├── shared/                      # 공용 모듈 (Kotlin)
│   ├── build.gradle.kts         # 멀티플랫폼 설정
│   └── src/
│       ├── commonMain/          # Android/iOS 공통 코드
│       │   └── kotlin/...
│       ├── androidMain/         # Android 전용 (JVM)
│       │   └── kotlin/...
│       └── iosMain/             # iOS 전용 (Kotlin/Native)
│           └── kotlin/...
├── androidApp/                  # Android 앱
│   └── src/main/kotlin/...
└── iosApp/                      # iOS 앱 (Xcode 프로젝트)
    ├── iosApp.xcodeproj
    └── iosApp/
        └── ContentView.swift (SwiftUI)
```

**핵심 용어 정리**

| 용어 | 뜻 |
|------|-----|
| `commonMain` | 모든 플랫폼이 공유하는 코드 |
| `androidMain` | JVM에서만 실행되는 코드 |
| `iosMain` | Kotlin/Native로 컴파일되어 iOS에서 실행되는 코드 |
| `expect`/`actual` | 공용 인터페이스 선언 / 플랫폼별 구현 (다음 챕터) |
| Kotlin/Native | Kotlin 코드를 iOS가 읽는 네이티브 코드로 빌드 |

---

## 3. iOS에서 KMP 정확히 어떻게 실행될까?

- Kotlin/Native가 commonMain + iosMain 코드를 **프레임워크(예: `Shared`)** 로 컴파일합니다.
- Xcode 프로젝트는 이 프레임워크를 링크하고, Swift에서 `import Shared` 로 호출합니다.

```swift
// SwiftUI (iosApp)
import SwiftUI
import Shared   // ← Kotlin 코드가 프레임워크로 제공됨

struct ContentView: View {
    // Greeting은 commonMain의 Kotlin 함수
    let message = Greeting().welcome()
    ...
}
```

```kotlin
// commonMain/kotlin/...
class Greeting {
    fun welcome(): String = "Hello from KMP!"
}
```

그러면 `Greeting().welcome()`이 Swift에서 그대로 보입니다. 이게 전부입니다.

---

## 4. 첫 KMP 프로젝트 만들기 (공식 템플릿)

### 방법 1: Kotlin Multiplatform 웹사이트에서 다운로드

https://kmp.jetbrains.com/ 에서:
1. 앱 이름(예: `KMPBasics`), 공유 모듈 이름, 패키지명 입력
2. iOS 프레임워크 옵션: `xcframework` 또는 직접 포함
3. Download 버튼 → 압축 해제 → Android Studio에서 열기

### 방법 2: Android Studio 마법사

Android Studio → `New Project` → `Kotlin Multiplatform App` 템플릿 선택 → 설정 진행

**iOS 앱 열기** (프로젝트에 이미 `iosApp/iosApp.xcodeproj`이 포함되어 있습니다):

```bash
cd iosApp
xed .        # Xcode로 열기
```

> iOS 시뮬레이터용 프레임워크가 표시되도록 Gradle이 자동으로 빌드 설정을 잡아줍니다.

### 방법 3: 공용 모듈의 의존성 (build.gradle.kts)

기본적인 `shared/build.gradle.kts` 살펴보기:

```kotlin
plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
    id("com.android.library")
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions.jvmTarget = "17"
        }
    }

    listOf(
        iosX64(),          // 시뮬레이터(Intel)
        iosArm64(),        // 실 디바이스(Apple Silicon)
        iosSimulatorArm64()// 시뮬레이터(Apple Silicon)
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "Shared"
            isStatic = true
        }
    }

    sourceSets {
        commonMain.dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.1")
            implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.0")
            implementation("io.ktor:ktor-client-core:3.1.0")
        }
        androidMain.dependencies {
            implementation("io.ktor:ktor-client-okhttp:3.1.0")
        }
        iosMain.dependencies {
            implementation("io.ktor:ktor-client-darwin:3.1.0")
        }
    }
}
```

---

## 5. 빌드 실행

```bash
# Android 라이브러리 빌드
./gradlew :shared:assembleDebug

# iOS 프레임워크 빌드 (Apple Silicon 시뮬레이터용)
./gradlew :shared:linkDebugFrameworkIosSimulatorArm64

# 모든 작업 확인
./gradlew tasks
```

> Xcode에서 iosApp을 실행하면 Xcode가 Gradle을 통해 프레임워크를 자동 빌드해줍니다.
> 처음 몇번의 빌드는 Gradle 의존성 다운로드로 오래 걸립니다.

---

## 6. 공용 코드에서 뭘 할 수 있는지 미리보기

KMP는 네이티브 API를 공용으로 사용하는 방법(expect/actual)을 통해 다음을 지원합니다:

- Kotlin 표준 라이브러리 (컬렉션, 문자열, 수학)
- kotlinx.coroutines (비동기)
- kotlinx.serialization (JSON)
- Ktor (HTTP 통신) ← 5.3
- SQLDelight (데이터베이스) ← 5.4
- Android 3rd party 라이브러리 중 공용 지원되는 것들

Swift에서 직접 사용하기 어려운 기능(UIKit 등)은 iosMain에서 처리하고,
공용 로직이 기대하는 동작만 expect로 선언하는 것이 정석입니다.

---

## 🛠 실습: 첫 KMP 앱 빌드 & 양 플랫폼 실행

**요구사항**

1. 공식 템플릿으로 `KMPBasics` 프로젝트 생성
2. `shared/src/commonMain`에 `Greeting` 클래스를 수정:
   - `fun greeting(name: String): String = "Hello, $name! KMP에서 왔어요"` 
3. Android 앱 실행: 공용 함수가 표시되는지 확인
4. Xcode에서 iosApp 실행: **동일한** 공용 함수가 SwiftUI에서 표시되는지 확인

**실습 힌트**

Android:
```kotlin
// androidApp/MainActivity.kt
setContent {
    Text(Remember { Greeting().greeting("Minho") })
}
```

iOS:
```swift
// iosApp/ContentView.swift
let message = Greeting().greeting(name: "Minho")
Text(message)
```

> Kotlin의 기본 인자 함수나 특정 데이터 타입은 Swift에서 이름이 바뀔 수 있습니다. 에러가 나면 `Greeting().greeting(name: "Minho")` 형태로 호출 시도.

---

## 7. 자주 겪는 문제

| 증상 | 해결 |
|------|------|
| Xcode에서 `No such module 'Shared'` | Xcode 빌드 스페이스에서 Build Phase의 스크립트가 Gradle 빌드를 먼저 실행하는지 확인 |
| 실 디바이스 빌드 실패 | `iosArm64` 타겟 빌드 + Xcode에서 시뮬레이터/기기 아키텍처 일치 확인 |
| Release 빌드에서 Kotlin 함수가 안 보임 | `isStatic = true` 유지, Xcode의 `Embed & Sign`이 아니라 "Do not embed"로 |
| 빌드가 계속 실패 | `./gradlew clean` 후 다시, 네트워크 확인 |

---

## 마무리 & 다음 챕터

KMP는 "공용 로직 + 양쪽 화면" 구조입니다.

다음 챕터 `02_expect_actual과_플랫폼_통합.md`에서 플랫폼마다 다르게 구현해야 하는 기능을 처리하는 `expect`/`actual`을 학습합니다.