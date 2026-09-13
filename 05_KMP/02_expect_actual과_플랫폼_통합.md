# 5.2 expect/actual과 플랫폼 통합

> 플랫폼마다 구현이 달라야 하는 기능을 expect(선언)/actual(구현)로 처리하고, iOS Swift에서 공용 코드와 데이터를 주고받는다.

---

## 학습 목표

- `expect`/`actual`의 개념을 이해한다.
- 시간, 디바이스 정보, 플랫폼 구분 등 대표적인 예를 구현한다.
- Kotlin 데이터를 Swift에서 읽고 쓰는 방법(직렬화, 상호운용성)을 이해한다.
- iOS ↔ 공용 코드 간 흐름을 정리한다.

---

## 1. expect / actual이란?

- `expect`: 공용(commonMain)에 "이런 기능이 필요하다"고 선언만 함
- `actual`: 플랫폼별(androidMain/iosMain)로 실제 구현

```kotlin
// commonMain
expect fun currentPlatform(): String

// androidMain
actual fun currentPlatform(): String = "Android (JVM)"

// iosMain
actual fun currentPlatform(): String = "iOS (Native)"
```

호출은 공용에서 그냥:

```kotlin
class Greeting {
    fun describe(): String = "Running on ${currentPlatform()}"
}
```

이렇게 플랫폼별 파편화 없이 공용 로직에서 서로 다른 능력을 사용할 수 있습니다.

---

## 2. 대표 예: 디바이스 정보 가져오기

### 공용 선언

```kotlin
// commonMain/Platform.kt
expect fun osVersion(): String
expect fun deviceName(): String
expect fun isSimulator(): Boolean
```

### Android 구현

```kotlin
// androidMain/Platform.android.kt
import android.os.Build

actual fun osVersion(): String = "Android ${Build.VERSION.RELEASE}"
actual fun deviceName(): String = Build.MODEL
actual fun isSimulator(): Boolean = false
```

### iOS 구현

```kotlin
// iosMain/Platform.ios.kt
import platform.UIKit.UIDevice
import platform.UIKit.UIScreen

actual fun osVersion(): String = UIDevice.currentDevice.systemVersion
actual fun deviceName(): String = UIDevice.currentDevice.name
actual fun isSimulator(): Boolean =
    UIDevice.currentDevice.model.contains("Simulator")
```

> Kotlin/Native는 iOS의 UIKit/Foundation API를 그대로 Kotlin 문법으로 호출합니다 (C/cpp 로 만든 브리징).

---

## 3. 기타 유용한 expect/actual 예제

### 현재 시간 (플랫폼 의존적 형식)

```kotlin
// commonMain
expect fun currentDateTime(): String

// androidMain
actual fun currentDateTime(): String =
    java.time.LocalDateTime.now().toString()

// iosMain
import platform.Foundation.NSDateFormatter
import platform.Foundation.NSDate
import platform.Foundation.dateWithTimeIntervalSince1970

actual fun currentDateTime(): String {
    val formatter = NSDateFormatter()
    formatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
    return formatter.stringFromDate(NSDate())
}
```

### 무작위 UID 생성

```kotlin
// commonMain
expect fun generateId(): String

// androidMain
actual fun generateId(): String = java.util.UUID.randomUUID().toString()

// iosMain
import platform.Foundation.NSUUID

actual fun generateId(): String = NSUUID().UUIDString
```

---

## 4. 파일 저장 (플랫폼 API 차이)

로컬 파일 IO는 자주 expect/actual의 대상이 됩니다.

```kotlin
// commonMain
expect class LocalStore {
    fun save(key: String, value: String)
    fun load(key: String): String?
    fun delete(key: String)
}
```

```kotlin
// androidMain
import android.content.Context
actual class LocalStore(private val context: Context) {
    private val prefs get() =
        context.getSharedPreferences("app", Context.MODE_PRIVATE)

    actual fun save(key: String, value: String) {
        prefs.edit().putString(key, value).apply()
    }
    actual fun load(key: String): String? = prefs.getString(key, null)
    actual fun delete(key: String) {
        prefs.edit().remove(key).apply()
    }
}
```

```kotlin
// iosMain
import platform.Foundation.NSUserDefaults

actual class LocalStore {
    actual fun save(key: String, value: String) {
        NSUserDefaults.standardUserDefaults.setObject(value, forKey = key)
    }
    actual fun load(key: String): String? =
        NSUserDefaults.standardUserDefaults.stringForKey(key)
    actual fun delete(key: String) {
        NSUserDefaults.standardUserDefaults.removeObjectForKey(key)
    }
}
```

> 간단한 설정값은 `russhwolf/multiplatform-settings` 라이브러리가 이 boilerplate를 대체합니다 (5.4에서 사용).

---

## 5. Kotlin 데이터를 Swift에서 사용하기

### 기본 타입은 그대로 노출

```kotlin
class Calculator {
    fun add(a: Int, b: Int): Int = a + b
    fun message(): String = "계산 완료"
}
```

```swift
let calc = Calculator()
calc.add(a: 3, b: 4)        // 7
calc.message()              // "계산 완료"
```

### 공용 모델을 Swift에서 그대로 사용

```kotlin
@Serializable
data class User(
    val id: String,
    val name: String,
    val email: String
) {
    fun displayName(): String = "$name ($email)"
}
```

```swift
let user = User(id: "1", name: "Minho", email: "minho@example.com")
user.displayName()
```

### 자주 겪는 이름 변환 규칙

| Kotlin | Swift로 노출 시 |
|--------|----------------|
| `fun add(a: Int, b: Int)` | `add(a:b:)` |
| 기본 인자 함수 | `add(a:b:)` (기본값 버전 `addDefault_mask_:` 형태로도 노출됨) |
| `suspend fun fetch()` | 컴파일러 설정에 따라 CompletionHandler 클로저 또는 async 작업으로 노출 |
| Companion object의 함수 | `Companion`을 통해 접근 |
| `Boolean` | `Bool` |
| `List<T>` | `[T]` (매핑됨) |
| `Map<K,V>` | `[K: V]` (키/값 매핑) |

---

## 6. suspend 함수를 Swift에서 호출

Kotlin의 suspend 함수는 iOS에서 기본적으로 **클로저(completion handler)** 형태로 노출됩니다.

```kotlin
class GreetingRepository {
    suspend fun loadGreeting(name: String): String {
        kotlinx.coroutines.delay(1000)
        return "안녕, $name!"
    }
}
```

Swift에서 호출:

```swift
let repo = GreetingRepository()
repo.loadGreeting(name: "Minho") { greeting, error ->
    if let error = error {
        print("오류: \(error)")
    } else {
        print(greeting ?? "")
    }
}
```

> 최신 Kotlin/Native 설정에서는 Swift 5.5+의 `async/await`로도 노출할 수 있습니다
> (`kotlin.native.binary.objcExportSuspendFunctionLaunchThreadRestriction` 등 설정을 조정하거나,
> 프레임워크를 Swift async로 매핑: `binaries.framework { exportForwardDeclarations = true }` 등).
> 일단 클로저 콜백으로 동작 확인이 우선입니다.

---

## 7. iOS에서 공용 코드 테스트 (make 아키텍처 확인)

빌드된 프레임워크를 직접 열어 노출된 API를 확인할 수 있습니다.

```bash
./gradlew :shared:linkDebugFrameworkIosSimulatorArm64
```

생성 경로: `shared/build/bin/iosSimulatorArm64/debugFramework/Shared.framework`

---

## 🛠 실습 1: 플랫폼 정보 화면 (expect/actual)

**요구사항** (KMPBasics 프로젝트 이어서)

1. `expect class PlatformInfo` 만들고 android/ios actual 구현:
   - `fun name(): String`
   - `fun version(): String`
   - `fun isTablet(): Boolean`
2. 공용 `Greeting.describe()`에서 이 정보를 조합해 문자열 반환
3. Android 앱 + iOS 앱 각각 실행해 **서로 다른** 결과가 나오는지 확인

**실습 힌트**

- iOS `isTablet`: `UIDevice.currentDevice.userInterfaceIdiom == UIUserInterfaceIdiomPad`
- Android 굳이 tablet 구분이 어려우면 `Build.MANUFACTURER` 등으로 대체해도 OK

---

## 🛠 실습 2: 설정 저장 공용 API (LocalStore)

**요구사항**

1. 위 `LocalStore` 예제를 실제 shared 모듈에 추가
2. Android: `Context`를 주입받아 SharedPreferences 사용
   - Android의 MainActivity에서 `LocalStore(applicationContext)` 생성해 전달
3. iOS: NSUserDefaults 사용
4. 공용 테스트 코드(commonTest)에서 저장/로드/삭제 동작 검증

---

## 🛠 실습 3: Kotlin에서 Swift/UIKit 대기 (간단 샘플)

**요구사항**

1. 공용 `ToUpperCaseUseCase` 작성: `suspend fun convert(text: String): String`  (500ms 후 대문자 반환)
2. iOS SwiftUI 화면: TextField 입력 → 버튼 클릭 → `convert` 콜백으로 결과를 화면에 표시
3. 같은 화면에서 하단에 `PlatformInfo().name()` 표시

**실습 힌트** — iOS ContentView

```swift
struct ContentView: View {
    @State private var input = ""
    @State private var output = "결과 대기중"

    var body: some View {
        VStack {
            TextField("입력", text: $input).textFieldStyle(.roundedBorder)
            Button("변환") {
                ToUpperCaseUseCase().convert(text: input) { result, error in
                    output = result ?? "오류"
                }
            }
            Text(output)
            Text(PlatformInfo().name())
        }
        .padding()
    }
}
```

---

## 마무리 & 다음 챕터

expect/actual로 플랫폼별 기능을 공용 로직에 통합했습니다.

다음 챕터 `03_공통_네트워킹_Ktor.md`에서 Ktor로 네트워크 계층을 공용으로 만듭니다.