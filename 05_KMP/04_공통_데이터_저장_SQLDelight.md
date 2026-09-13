# 5.4 KMP 공통 데이터 저장: SQLDelight

> SQLDelight로 공용 로컬 데이터베이스를 다루고, iOS/Android 동일한 SQL와 코드로 저장·조회한다.

---

## 학습 목표

- SQLDelight의 개념과 .sq 파일 문법을 이해한다.
- 공용 모델(Query/CRUD)을 Kotlin 코드로 생성하는 흐름을 안다.
- Android와 iOS에서 실제로 사용자 목록을 저장·조회한다.
- multiplatform-settings로 설정 값도 공용화한다.

---

## 1. SQLDelight란?

- Kotlin으로 플랫폼별 DB(SQLite)를 추상화한 라이브러리
- `.sq` 파일에 SQL 스키마·쿼리 작성 → 빌드가 **타입 안전 Kotlin 코드**를 생성
- Android: 안드로이드 SQLite 드라이버 사용
- iOS: SQLite(Native) 드라이버 사용

```
shared/src/commonMain/sqldelight/TodoDatabase.sq   ← SQL 작성
        │  Gradle 빌드
        ▼
생성된 타입 안전 쿼리 클래스 (예: TodoQueries)
```

---

## 2. 설정 (build.gradle.kts)

```kotlin
plugins {
    kotlin("multiplatform")
    id("app.cash.sqldelight")
}

sqldelight {
    databases {
        create("AppDatabase") {
            packageName.set("com.example.db")
        }
    }
}

commonMain.dependencies {
    implementation("app.cash.sqldelight:runtime:2.0.2")
    implementation("app.cash.sqldelight:coroutines-extensions:2.0.2")
}
androidMain.dependencies {
    implementation("app.cash.sqldelight:android-driver:2.0.2")
}
iosMain.dependencies {
    implementation("app.cash.sqldelight:native-driver:2.0.2")
}
```

---

## 3. 스키마 작성 (.sq 파일)

경로: `shared/src/commonMain/sqldelight/com/example/db/TodoDatabase.sq`

```sql
CREATE TABLE todo (
    id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    completed INTEGER AS Boolean NOT NULL DEFAULT 0,
    created_at INTEGER AS Long NOT NULL
);

insertTodo:
INSERT INTO todo(title, completed, created_at)
VALUES (?, ?, ?);

selectAll:
SELECT * FROM todo ORDER BY created_at DESC;

selectById:
SELECT * FROM todo WHERE id = ?;

updateCompleted:
UPDATE todo SET completed = ? WHERE id = ?;

deleteTodo:
DELETE FROM todo WHERE id = ?;

selectByCompleted:
SELECT * FROM todo WHERE completed = ?;
```

- 타입 매핑: `INTEGER AS Boolean`, `INTEGER AS Long` 은 자동 변환
- 쿼리 이름 작성 → 자동으로 `AppDatabaseQueries`에 함수가 생성됩니다.

---

## 4. 드라이버 (플랫폼별 READY)

### Android (androidMain)

```kotlin
// androidMain/DatabaseDriverFactory.android.kt
import android.content.Context
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.android.AndroidSqliteDriver

class DatabaseDriverFactory(private val context: Context) {
    fun createDriver(): SqlDriver {
        return AndroidSqliteDriver(
            schema = AppDatabase.Schema,
            context = context,
            name = "app.db"
        )
    }
}
```

### iOS (iosMain)

```kotlin
// iosMain/DatabaseDriverFactory.ios.kt
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.native.NativeSqliteDriver

class DatabaseDriverFactory {
    fun createDriver(): SqlDriver {
        return NativeSqliteDriver(
            schema = AppDatabase.Schema,
            "app.db"
        )
    }
}
```

---

## 5. 공용 DB 사용 (commonMain)

공용 코드는 생성된 타입 안전 API를 사용합니다:

```kotlin
// commonMain/TodoLocalDataSource.kt
import app.cash.sqldelight.coroutines.asFlow
import kotlinx.coroutines.flow.Flow

class TodoLocalDataSource(private val db: AppDatabase) {

    fun observeTodos(): Flow<List<Todo>> =
        db.todoQueries
            .selectAll()
            .asFlow()
            .mapToList()

    fun insert(title: String) {
        db.todoQueries.insertTodo(
            title = title,
            completed = false,
            created_at = System.currentTimeMillis()
        )
    }

    fun toggleCompleted(id: Long) {
        db.todoQueries.updateCompleted(completed = true, id = id)
    }

    fun delete(id: Long) {
        db.todoQueries.deleteTodo(id)
    }
}
```

> `asFlow() + mapToList()`로 DB 변경을 Flow로 관찰 → UI 자동 갱신.

---

## 6. Android에서 조립하기

```kotlin
// androidApp/MainActivity or App
class TodoApplication : Application() {
    lateinit var dataSource: TodoLocalDataSource
        private set

    override fun onCreate() {
        super.onCreate()
        val driver = DatabaseDriverFactory(this).createDriver()
        val db = AppDatabase(driver)
        dataSource = TodoLocalDataSource(db)
    }
}
```

```kotlin
@Composable
fun TodoScreen(dataSource: TodoLocalDataSource) {
    val todos by dataSource.observeTodos()
        .collectAsState(initial = emptyList())

    LazyColumn {
        items(todos, key = { it.id }) { todo ->
            Row(
                modifier = Modifier.fillMaxWidth().clickable {
                    dataSource.toggleCompleted(todo.id)
                }
            ) {
                Text(if (todo.completed) "✅" else "⬜")
                Text(todo.title)
            }
        }
    }
}
```

---

## 7. iOS에서 조립 (Swift)

SQLDelight 드라이버는 Kotlin 쪽에서 만들고 Swift에 노출합니다.

```kotlin
// iosMain, 공용 진입점으로 프레임워크 노출
class DatabaseSetup {
    companion object {
        fun createDataSource(): TodoLocalDataSource {
            val db = AppDatabase(DatabaseDriverFactory().createDriver())
            return TodoLocalDataSource(db)
        }
    }
}
```

```swift
// ContentView.swift
let dataSource = DatabaseSetup.createDataSource()

// Flow를 Swift에서 관찰하거나, 필요시 suspend로 한 번에 가져오기
Task {
    // Flow 리스트를 예시로 한 번 호출
    // 실제로는 completion 핸들러 형태로 내려옴
}
```

> Flow를 iOS에서 직접 collect 하려면 추가 브리징이 필요합니다. 간단한 예시로는 suspend 함수 하나에
> 목록을 담은 배열을 돌려주는 `loadOnce()` 같은 편의 함수를 iOS에 제공하는 방법을 권장합니다.

```kotlin
class TodoLocalDataSource {
    suspend fun getTodosOnce(): List<Todo> =
        db.todoQueries.selectAll().executeAsList()
}
```

```swift
Task {
    let todos = dataSource.getTodosOnce { result, error in
        // 사용
    }
}
```

---

## 8. multiplatform-settings (설정 값 공용화)

UserDefaults/SharedPreferences를 래핑하는 라이브러리:
`com.russhwolf:multiplatform-settings:1.3.0`

```kotlin
// commonMain
import com.russhwolf.settings.Settings
import com.russhwolf.settings.ExperimentalSettingsApi

@OptIn(ExperimentalSettingsApi::class)
class AppSettings(settings: Settings) {
    private val settings = settings

    var nickname: String
        get() = settings.getString("nickname", "게스트")
        set(value) { settings.putString("nickname", value) }

    var isDarkMode: Boolean
        get() = settings.getBoolean("dark", false)
        set(value) { settings.putBoolean("dark", value) }

    var sessionCount: Int
        get() = settings.getInt("sessions", 0)
        set(value) { settings.putInt("sessions", value) }
}
```

플랫폼 드라이버 직접 생성:

```kotlin
// androidMain
val settings = SharedPreferencesSettings(
    context.getSharedPreferences("app", Context.MODE_PRIVATE)
)

// iosMain
val settings = NSUserDefaultsSettings(NSUserDefaults.standardUserDefaults)
```

---

## 🛠 실습 1: 메모장 DB 만들기

**요구사항**

1. `Note.sq` 작성: `id`, `title`, `content`, `created_at`
2. 쿼리: selectAll(최신순), insertNote, deleteNote, searchByTitle(패턴 매칭)
3. 공용 `NoteDataSource` 클래스 작성 (insert/delete/observe)
4. Android + iOS 양쪽에서 드라이버 연결

---

## 🛠 실습 2: 오프라인 우선 할 일 앱 (Ktor + SQLDelight 통합)

**요구사항**

1. 네트워크: `TodoRepository.fetchTodos()` (5.3의 것 재사용)
2. 로컬: `TodoLocalDataSource` 
3. **Sync UseCase**:
   - 서버에서 목록 가져오기 → 성공 시 로컬 전체 저장
   - 실패 시 → 로컬 목록 반환 (오프라인 우선)
4. Android 화면 구현 + 공용 로직은 commonTest로 검증

**실습 힌트**

```kotlin
class SyncUseCase(
    private val remote: TodoRemote,
    private val local: TodoLocalDataSource
) {
    suspend fun sync(): List<Todo> {
        return when (val r = remote.fetchTodos()) {
            is NetworkResult.Success -> {
                local.replaceAll(r.data)
                r.data
            }
            is NetworkResult.Failure -> local.loadAll()
        }
    }
}
```

로컬에 `replaceAll` 추가:

```kotlin
fun replaceAll(todos: List<Todo>) {
    db.todoQueries.deleteAll()
    todos.forEach { db.todoQueries.insertTodo(...) }
}
```

---

## 🛠 실습 3: 설정 저장 ViewModel 통합

**요구사항**

1. `AppSettings`를 Android/iOS에 연결
2. 다크모드 토글, 세션 카운트 등 저장되는지 확인
3. `Flow`로 설정 변화 관찰 가능한지 실험 (multiplatform-settings의 `Setting` API 사용)

**실습 힌트**

```kotlin
import com.russhwolf.settings.observable.ObservableSettings

val nickname: Setting<String> = settings.getStringSetting("nickname", "게스트")
// nickname.value 읽기, nickname.asFlow() 관찰
```

---

## 마무리 & 다음 챕터

공용 DB(SQLDelight)와 설정(multiplatform-settings)으로 로컬 저장까지 공용화했습니다.

이제 마지막 KMP 챕터 `05_Compose_Multiplatform.md`로 UI까지 공유하는 Compose Multiplatform을 학습합니다.