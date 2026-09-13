# 4.1 Jetpack Compose 입문: 첫 Android 앱

> Jetpack Compose로 첫 안드로이드 앱을 만들고, 선언형 UI의 기본을 학습한다.

---

## 학습 목표

- Android 프로젝트 구조와 Gradle을 이해한다.
- Compose의 `Composable` 함수와 기본 컴포넌트를 사용할 수 있다.
- `remember`, `mutableStateOf`로 상태를 관리한다.
- `LazyColumn`과 `Material3` 컴포넌트를 사용해 화면을 구성한다.

---

## 1. 프로젝트 만들기

### 절차

1. Android Studio 실행 → `New Project`
2. 템플릿: **Empty Activity** (기본은 Empty Views Activity가 아니게 주의)
3. Project 정보:
   - Name: `MyFirstComposeApp`
   - Package: `com.example.myfirst`
   - Minimum SDK: `API 26` (또는 기본값)
4. `Finished` 후 Gradle 동기화 완료 대기
5. 에뮬레이터 실행 후 ▶ Run

---

## 2. 프로젝트 구조

```
app/src/main/
├── java/com/example/myfirst/
│   ├── MainActivity.kt
│   └── ui/theme/ ... (테마 파일)
└── res/ ... (리소스)
```

`MainActivity.kt`:

```kotlin
package com.example.myfirst

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.Text

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Text("Hello, Compose!")
        }
    }
}
```

- `setContent { }` 안이 전부 UI = **선언형 UI** (SwiftUI와 동일 철학)

---

## 3. Composable 함수

UI 조각을 함수로 선언합니다. `@Composable` 애노테이션을 붙입니다.

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "안녕, $name!")
}

@Composable
fun MainScreen() {
    Greeting(name = "Minho")
}
```

> `@Composable` 함수는 상태를 읽을 때 자동으로 "내가 바뀌면 다시 그려짐" 처리됩니다. SwiftUI의 `View`와 같습니다.

---

## 4. 기본 컴포넌트 (Material3)

### Text

```kotlin
@Composable
fun BasicText() {
    Text(
        text = "SwiftUI와 비슷한 선언형 UI",
        fontSize = 20.sp,
        fontWeight = FontWeight.Bold,
        color = Color.Blue
    )
}
```

### Button

```kotlin
@Composable
fun MyButton() {
    Button(onClick = {
        println("눌렸다!")
    }) {
        Text("눌러보세요")
    }
}
```

### Column / Row / Box (SwiftUI의 V/H/ZStack 대응)

```kotlin
Column(
    modifier = Modifier.fillMaxSize(),
    horizontalAlignment = Alignment.CenterHorizontally,
    verticalArrangement = Arrangement.spacedBy(16.dp)
) {
    Text("위")
    Row {
        Text("왼쪽")
        Spacer(modifier = Modifier.width(20.dp))
        Text("오른쪽")
    }
    Box(
        modifier = Modifier.size(100.dp).background(Color.Yellow)
    ) {
        Text("겹침", modifier = Modifier.align(Alignment.Center))
    }
}
```

---

## 5. 상태 관리

### remember + mutableStateOf (SwiftUI @State 대응)

```kotlin
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text("카운트: $count", fontSize = 40.sp)

        Row {
            Button(onClick = { if (count > 0) count-- }) {
                Text("-")
            }
            Button(onClick = { count++ }) {
                Text("+")
            }
        }
    }
}
```

- `by remember { mutableIntStateOf(0) }` — 상태가 바뀌면 count를 쓰는 화면만 재구성
- SwiftUI와 같은 단방향 데이터 흐름

### StateXxx 종류

| SwiftUI | Compose |
|---------|---------|
| `@State` | `remember { mutableStateOf(...) }` |
| `@Binding`/`$` | 값 전달 + 콜백 |
| `@Observable` | `StateFlow` + `collectAsState()` |
| `@Query` | `Room` / `Flow` + collect |

---

## 6. 리스트: LazyColumn

```kotlin
@Composable
fun FruitList() {
    val fruits = listOf("사과", "바나나", "오렌지", "포도", "키위")

    LazyColumn {
        items(fruits) { fruit ->
            Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
                Text(
                    text = fruit,
                    modifier = Modifier.padding(16.dp),
                    fontSize = 18.sp
                )
            }
        }
    }
}
```

### 데이터 클래스 + Index

```kotlin
data class Song(val id: Int, val title: String, val artist: String)

@Composable
fun SongList(songs: List<Song>) {
    LazyColumn {
        items(songs, key = { it.id }) { song ->   // key로 고유성
            SongRow(song)
        }
    }
}

@Composable
fun SongRow(song: Song) {
    Row(
        modifier = Modifier.fillMaxWidth().padding(12.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Text(song.title)
        Text(song.artist, style = MaterialTheme.typography.bodySmall)
    }
}
```

---

## 7. 텍스트 입력 (TextField)

```kotlin
@Composable
fun ChatInput() {
    var text by remember { mutableStateOf("") }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("메시지") },
        singleLine = true
    )

    Button(onClick = { println("전송: $text"); text = "" }) {
        Text("전송")
    }
}
```

---

## 8. Material3 테마 적용

```kotlin
setContent {
    MaterialTheme(colorScheme = lightColorScheme()) {
        MainScreen()
    }
}
```

기본 템플릿의 `ui/theme/Theme.kt`가 Material3 테마를 이미 제공합니다.

---

## 9. 뷰모델 + StateFlow 패턴 (Android 정석)

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class MainViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count

    fun increment() {
        viewModelScope.launch {
            _count.value += 1
        }
    }
}
```

```kotlin
@Composable
fun CountScreen(viewModel: MainViewModel) {
    val count by viewModel.count.collectAsState()   // 화면에 구독

    Column {
        Text("$count")
        Button(onClick = { viewModel.increment() }) {
            Text("+")
        }
    }
}
```

---

## 🛠 실습 1: 할 일 목록 앱

**요구사항**

1. `data class Todo(val id: Int, val title: String, var done: Boolean)`
2. 상단 TextField + "추가" 버튼 → `mutableListOf<Todo>` 에 append
3. `LazyColumn`에 `items(todos, key = { it.id })`로 표시
4. 체크 버튼은 `Checkbox`로 done 토글, 완료 항목은 취소선
5. 스와이프 삭제(`SwipeToDismiss`) 또는 longPress 제거 중 택일
6. 완료 개수를 상단에 표시

**실습 힌트**

- 리스트 상태는 `var todos by remember { mutableStateOf(listOf<Todo>()) }`
- 삭제: `todos = todos.filterNot { it.id == target.id }`
- 취소선: `.textDecoration(TextDecoration.LineThrough)`

---

## 🛠 실습 2: 하드웨어 반응형 프로필 카드

**요구사항**

1. `Box` + 황금 비율 배경으로 프로필 카드 UI 구성
2. 클릭 시 확대 `AnimatedVisibility` 또는 `scale` 변화 (animateContentSize)
3. 입력된 닉네임을 미리보기로 동시에 표시
4. 다크모드 테마(`isSystemInDarkTheme()`) 연동 토글

**실습 힌트**

```kotlin
val dark = isSystemInDarkTheme()
MaterialTheme(colorScheme = if (dark) darkColorScheme() else lightColorScheme())
```

---

## 🛠 실습 3: 지금까지 종합 - 음악 메모 앱

1. 노래 제목/가사 메모 `data class Note(id, title, lyric, date)`
2. 목록 + 상세(LazyColumn/Navigation) 
   - Compose에서 화면 전환은 `NavHost` (Navigation 라이브러리) 사용
3. 검색 필터: `remember(text)` 로 title/lyric contains 필터

---

## 마무리 & 다음 챕터

이제 Swift(2단계)와 Kotlin/Android(3~4단계)를 모두 파악했습니다.

다음은 메인 캐롯! `05_KMP/01_KMP_개념과_프로젝트_구조.md` 로 Kotlin Multiplatform 학습을 시작합니다.