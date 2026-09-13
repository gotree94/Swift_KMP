# 3.3 코루틴과 Flow (Kotlin 비동기)

> Kotlin의 핵심 비동기 프레임워크인 코루틴과, 데이터 스트림인 Flow를 학습한다. KMP의 공용 코드의 핵심이 된다.

---

## 학습 목표

- 코루틴의 개념과 `suspend`, `launch`, `async`를 이해한다.
- `Dispatchers`로 스레드를 제어할 수 있다.
- `Flow`, `StateFlow`, `SharedFlow`의 차이를 이해한다.
- Swift의 async/await와 비교하며 개념을 단단히 한다.

---

## 1. 코루틴이란?

가볍고 연쇄되는 비동기 작업 단위입니다. 스레드를 만드는 비용 없이 일시정지/재개합니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("main 시작")

    launch {                    // 새 코루틴 시작 (비동기)
        delay(1000)             // 1초 대기 (스레드 블록 아님)
        println("launch 내부")
    }

    println("main 계속 진행")
}
```

출력 순서: `main 시작` → `main 계속 진행` → `launch 내부`

**핵심**: `delay`는 스레드를 점유하지 않습니다. suspend 함수 사이에서 아무 일도 안 하는 동안 다른 코루틴이 실행될 수 있습니다.

---

## 2. suspend 함수

`delay`, `withContext`, 네트워크 등 시간이 걸리는 일을 하는 함수는 **suspend**로 표시합니다.

```kotlin
suspend fun fetchUser(id: Int): String {
    delay(1000)                 // 가짜 네트워크 지연
    return "user-$id"
}

fun main() = runBlocking {
    val result = fetchUser(1)   // suspend 지점
    println(result)
}
```

- suspend 함수는 suspend 지점에서 **일시정지** 후, 결과가 오면 **재개**됩니다.
- Swift의 `async/await`와 대응: Kotlin `suspend fun` = Swift `async func`, 호출부에 `await` 필요 없음.

---

## 3. async / await

병렬 작업 결과를 모을 때 사용합니다.

```kotlin
suspend fun loadProfile(id: Int): String {
    delay(1000)
    return "프로필 $id"
}

suspend fun loadPosts(count: Int): String {
    delay(1500)
    return "게시글 $count 개"
}

fun main() = runBlocking {
    // 병렬 실행
    val a = async { loadProfile(7) }
    val b = async { loadPosts(20) }

    println("${a.await()} + ${b.await()}")   // 둘 다 끝나기를 기다림
}
```

- `async` = 결과가 필요한 병렬 작업, `launch` = 결과 필요 없는 화재성 작업
- 총 소요시간: 2.5초 (병렬이라 1000+1500이 합쳐짐)

---

## 4. Dispatchers (스레드 제어)

```kotlin
fun main() = runBlocking {
    launch(Dispatchers.IO) {        // 네트워크/DB 작업
        println("IO 스레드에서 실행")
    }

    launch(Dispatchers.Main) {      // UI 작업 (안드로이드에서 사용)
        println("메인 스레드")
    }

    launch(Dispatchers.Default) {   // CPU 연산
        println("기본 스레드")
    }
}

// 특정 거대 연산을 다른 스레드로
suspend fun heavyWork() = withContext(Dispatchers.Default) {
    (1..1_000_000).sum()            // blocking 연산
}
```

---

## 5. Flow (데이터 스트림)

여러 값을 순차적으로 내보내는 스트림입니다. Swift의 AsyncSequence / Combine Publisher와 유사.

```swift
import kotlinx.coroutines.flow.*

fun counterFlow(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(300)
        emit(i)                  // 값 방출
    }
}

fun main() = runBlocking {
    counterFlow()
        .collect { value ->      // 구독/소비
            println("받음: $value")
        }
}
```

### Flow 연산자 (Swift Combine의 map/filter와 유사)

```kotlin
fun main() = runBlocking {
    (1..10).asFlow()
        .filter { it % 2 == 0 }            // 짝수만
        .map { it * 10 }                    // 변환
        .take(3)                            // 3개까지
        .collect { println(it) }            // 20, 40, 60
}
```

### 오류 처리

```kotlin
fun main() = runBlocking {
    flow {
        emit(1)
        throw RuntimeException("실패")
    }
        .catch { e -> println("오류: ${e.message}") }
        .onCompletion { println("완료") }
        .collect { println(it) }
}
```

---

## 6. StateFlow / SharedFlow

상태(State)를 관찰 가능하게 만드는 Flow입니다. SwiftUI의 @State/@Observable, Combine의 @Published 역할입니다.

```kotlin
class CounterStore {
    private val _count = MutableStateFlow(0)   // 비공개 실제 상태
    val count: StateFlow<Int> = _count          // 공개 관찰용

    fun increment() {
        _count.value += 1
    }
}

fun main() = runBlocking {
    val store = CounterStore()

    val job = launch {
        store.count.collect { value ->
            println("카운트: $value")
        }
    }

    store.increment()   // 카운트: 1
    store.increment()   // 카운트: 2

    delay(100)
    job.cancel()
}
```

### StateFlow vs SharedFlow

| 구분 | StateFlow | SharedFlow |
|------|-----------|------------|
| 역할 | 최신 상태 저장, 관찰자에게 즉시 전달 | 이벤트 전달 (구독한 순간부터) |
| 버퍼 | 최신 1개 | 설정한 버퍼(기본 64) |
| 중복 최신값 | 중복 방출 안 함 | 연속으로 방출 가능 |
| 용도 | UI 상태 | 클릭 이벤트, 토스트 |

```kotlin
class EventBus {
    private val _events = MutableSharedFlow<String>()
    val events: SharedFlow<String> = _events

    suspend fun send(message: String) {
        _events.emit(message)
    }
}
```

---

## 7. 뷰모델에서의 실전 패턴 (Android에서 재사용)

다음 챕터(Android)에서 쓸 공통 패턴입니다.

```kotlin
sealed class UiState {
    data class Loading(val message: String = "로딩중") : UiState()
    data class Success(val data: List<String>) : UiState()
    data class Error(val message: String) : UiState()
}

class MainViewModel {
    private val _state = MutableStateFlow<UiState>(UiState.Loading())
    val state: StateFlow<UiState> = _state

    suspend fun load() {
        _state.value = UiState.Loading("불러오는 중...")
        try {
            delay(1000)   // 실제 네트워크 대체
            _state.value = UiState.Success(listOf("A", "B", "C"))
        } catch (e: Exception) {
            _state.value = UiState.Error(e.message ?: "오류")
        }
    }
}
```

---

## 🛠 실습 1: 병렬 API 폴링

`coroutines_practice.kt`로 제작하세요.

**요구사항**

1. `simulateRequest(name: String, ms: Long): String` suspend 로 구현
2. `async` 3개를 동시 실행해 `"A/CE/B"` 같은 이름 3개를 모두 이으면 결과 출력
3. `measureTimeMillis`로 걸린 시간 출력 → 병렬이 2초가 아닌 최댓값만 걸리는지 확인

**실습 힌트**

```kotlin
import kotlin.system.measureTimeMillis

val time = measureTimeMillis {
    // async X3 + await
}
println("총 $time ms")
```

---

## 🛠 실습 2: 실제 작업: 텍스트 찾기 (Flow + Debounce)

**요구사항**

1. `fun typingFlow(): Flow<String>`  = "쿠", "쿠키", "쿠키가", "쿠키가 필요해" 순 delay 300ms로 emit
2. 이 Flow에 `.debounce(400)` 적용: 빠르게 연속 입력된 것은 하나만 남도록
3. 각 발행마다 현재까지의 값(kotlinx 없으면 간단 구현)에서 `"쿠키"` 포함 여부 검사

**실습 힌트**

```kotlin
typingFlow()
    .debounce(400)
    .collect { s -> println("트리거: $s") }
```

---

## 🛠 실습 3: 앱 상태 관리 (StateFlow UI State)

**요구사항**

1. `sealed class UiState` (Loading / Success / Error)
2. `GameStore`: 
   - `MutableStateFlow<UiState>` + `fun start()`, `fun play()`, `fun fail(message: String)`
3. `main`에서 collect하며 상태 변화를 출력
4. `start()` 후 500ms 뒤 Success("게임 시작"), play() 호출마다 점수 갱신

---

## 실습 풀이

<details>
<summary>실습 1 정답</summary>

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

suspend fun simulateRequest(name: String, ms: Long): String {
    delay(ms)
    return name
}

fun main() = runBlocking {
    val time = measureTimeMillis {
        val r1 = async { simulateRequest("A", 1000) }
        val r2 = async { simulateRequest("B", 2000) }
        val r3 = async { simulateRequest("C", 500) }
        println("결과: ${r1.await()}${r2.await()}${r3.await()}")
    }
    println("총 시간: ${time}ms (병렬이면 약 2000ms)")
}
```
</details>

---

## 다음 챕터로

코루틴과 Flow는 KMP 공용 로직과 Android UI의 근간입니다.

다음 챕터 `04_Android/01_JetpackCompose_입문.md` 에서 Jetpack Compose로 첫 안드로이드 화면을 만듭니다.