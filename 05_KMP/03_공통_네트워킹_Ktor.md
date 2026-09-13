# 5.3 KMP 공통 네트워킹: Ktor Client

> Ktor Client로 iOS/Android 공통으로 쓸 네트워크 계층을 만들고, JSON 직렬화와 결과 처리를 배운다.

---

## 학습 목표

- Ktor Client의 엔진(OkHttp/Darwin)을 이해한다.
- REST API 호출(GET/POST)을 공용 모듈에서 구현한다.
- kotlinx.serialization으로 JSON을 모델화한다.
- 오류·타임아웃·로깅을 공용으로 처리한다.

---

## 1. Ktor 클라이언트 개요

Ktor는 JetBrains의 멀티플랫폼 HTTP 클라이언트입니다.

```
공용 코드(commonMain): Ktor 공용 API
      ├── Android 런타임 → Engine: OkHttp   (androidMain 의존성)
      └── iOS 런타임     → Engine: Darwin    (iosMain 의존성)
```

같은 공용 코드가 플랫폼 엔진을 자동으로 골라 사용합니다.

### 의존성 (shared/build.gradle.kts)

```kotlin
commonMain.dependencies {
    implementation("io.ktor:ktor-client-core:3.1.0")
    implementation("io.ktor:ktor-client-content-negotiation:3.1.0")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.1.0")
    implementation("io.ktor:ktor-client-logging:3.1.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.0")
}
androidMain.dependencies {
    implementation("io.ktor:ktor-client-okhttp:3.1.0")
}
iosMain.dependencies {
    implementation("io.ktor:ktor-client-darwin:3.1.0")
}
```

---

## 2. 클라이언트 만들기 (공용)

```kotlin
// commonMain/ApiClient.kt
import io.ktor.client.*
import io.ktor.client.engine.*
import io.ktor.client.plugins.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.plugins.logging.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json

fun createHttpClient() = HttpClient {
    // JSON 설정
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            isLenient = true
        })
    }
    // 타임아웃
    install(HttpTimeout) {
        requestTimeoutMillis = 15_000
        connectTimeoutMillis = 10_000
    }
    // 로깅
    install(Logging) {
        logger = Logger.DEFAULT
        level = LogLevel.INFO
    }
}
```

> 클라이언트 하나를 싱글턴으로 재사용하는 게 일반적입니다.

---

## 3. 모델과 데이터 클래스 (직렬화)

```kotlin
// commonMain/Models.kt
import kotlinx.serialization.Serializable

@Serializable
data class Todo(
    val id: Int,
    val userId: Int? = null,
    val title: String,
    val completed: Boolean = false
)
```

- `jsonplaceholder.typicode.com/todos` 무료 API 사용

---

## 4. Repository: 공용 네트워크 계층

```kotlin
// commonMain/TodoRepository.kt
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.request.*
import io.ktor.http.*

class TodoRepository(private val client: HttpClient = createHttpClient()) {

    private val baseUrl = "https://jsonplaceholder.typicode.com/todos"

    suspend fun fetchTodos(): List<Todo> {
        val response = client.get(baseUrl)
        return response.body()   // 직렬화 자동
    }

    suspend fun fetchTodo(id: Int): Todo {
        return client.get("$baseUrl/$id").body()
    }

    suspend fun createTodo(todo: Todo): Todo {
        val response = client.post(baseUrl) {
            contentType(ContentType.Application.Json)
            setBody(todo)
        }
        return response.body()
    }

    suspend fun updateCompleted(id: Int, completed: Boolean): Todo {
        val response = client.patch("$baseUrl/$id") {
            contentType(ContentType.Application.Json)
            setBody(mapOf("completed" to completed))
        }
        return response.body()
    }
}
```

> `request.body<T>()`는 Content-Type 200이면 자동 역직렬화, 오류면 throw.

---

## 5. 오류 처리 공용 로직

```kotlin
// commonMain/NetworkResult.kt
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Failure(val message: String, val code: Int? = null) : NetworkResult<Nothing>()
}

class TodoAPI(val client: HttpClient = createHttpClient()) {

    suspend fun fetchTodos(): NetworkResult<List<Todo>> {
        return try {
            val todos = client.get("https://jsonplaceholder.typicode.com/todos").body<List<Todo>>()
            NetworkResult.Success(todos)
        } catch (e: Exception) {
            NetworkResult.Failure(e.message ?: "네트워크 오류")
        }
    }
}
```

- 실패를 예외가 아닌 **값**으로 다루면 UI에서 표시가 쉽습니다.
- `sealed class` + when은 Android/iOS 어느 쪽에서든 상태 처리 패턴으로 쓰입니다.

---

## 6. iOS(Swift)에서 호출하기

suspend 함수는 클로저 콜백으로 노출됩니다:

```swift
let api = TodoAPI()

api.fetchTodos { result, error in
    if let error = error {
        print("실패: \(error)")
        return
    }
    // result는 List<Todo> → Swift 배열로 매핑
    for todo in result ?? [] {
        print("\(todo.id): \(todo.title)")
    }
}
```

`NetworkResult`를 그대로 노출할 수도 있지만(sealed 클래스), 콜백 시그니처 권장:

실제 KMP에서 코드는 다음처럼 생깁니다.

```swift
api.fetchTodos { result, error in
    if let error {
        state = .error(error.localizedDescription)
    } else {
        state = .loaded(result)
    }
}
```

---

## 7. 공용 테스트 (commonTest)

```kotlin
// shared/src/commonTest/kotlin/TodoApiTest.kt
import kotlinx.coroutines.test.runTest
import kotlin.test.*

class TodoApiTest {
    @Test
    fun fetchTodos_returnsNonEmpty() = runTest {
        val api = TodoAPI()
        val result = api.fetchTodos()
        assertTrue(result is NetworkResult.Success)
        assertTrue((result as NetworkResult.Success).data.isNotEmpty())
    }
}
```

실행:

```bash
./gradlew :shared:testDebugUnitTest
```

---

## 🛠 실습 1: 영화/책 목록 뷰모델 만들기

**요구사항**

공개 API 중 아무거나 하나 골라 공용 조회 함수를 만드세요. (예시 추천: 대량의 무료 더미 API)

1. `data class Post(id, title, body)` (jsonplaceholder)
2. `PostRepository.posts(page, limit)` 공용 구현
3. `NetworkResult`로 감싸 반환
4. commonTest에서 성공 케이스를 확인

**실습 힌트**

```kotlin
suspend fun fetchPosts(): NetworkResult<List<Post>> {
    val url = "https://jsonplaceholder.typicode.com/posts?_limit=10"
    ...
}
```

---

## 🛠 실습 2: 앱에 연동 (Android 화면)

**요구사항**

1. Android 앱에 `LazyColumn` + `collectAsState` 상태 연결
2. `UiState`: Loading / Success(posts) / Error
3. 화면 진입 시 공용 `PostRepository.fetchPosts()` 호출
4. 성공 시 목록, 실패 시 에러 + 재시도 버튼 표시

**실습 힌트** (AndroidViewModel + Compose)

```kotlin
class PostsViewModel : ViewModel() {
    private val repo = PostRepository()
    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state = _state

    fun load() {
        viewModelScope.launch {
            _state.value = UiState.Loading
            when (val result = repo.fetchPosts()) {
                is NetworkResult.Success -> _state.value = UiState.Success(result.data)
                is NetworkResult.Failure -> _state.value = UiState.Error(result.message)
            }
        }
    }
}
```

참고: Compose도 동일한 sealed class 상태를 그대로 사용합니다.

---

## 🛠 실습 3: iOS 연동 & UI

**요구사항**

1. iOS SwiftUI 화면에 같은 요청 결과 표시
2. `@State` 기반 ViewModel 클래스(ObservableObject) 또는 단순 `@State` 사용
3. 로딩 시 ProgressView, 실패 시 에러 + 재시도
4. 공용 코드가 **커피(코틀린)로 만든 로직**임을 확인: Android와 동일 응답

**실습 힌트**

```swift
class PostsViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var errorMessage: String?

    func load() {
        PostRepository().fetchPosts { result, error in
            DispatchQueue.main.async {
                if let result = result {
                    self.posts = result
                }
                self.errorMessage = error?.localizedDescription
            }
        }
    }
}
```

---

## 마무리 & 다음 챕터

Ktor로 서버 통신을 공용화했습니다. Android/iOS 양쪽이 같은 코드를 호출합니다.

다음 챕터 `04_공통_데이터_저장_SQLDelight.md`에서 데이터베이스를 공용으로 다룹니다.