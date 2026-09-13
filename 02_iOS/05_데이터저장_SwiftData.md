# 2.5 데이터 저장: SwiftData

> 앱이 종료되어도 데이터가 사라지지 않도록, 기기에 데이터를 저장하고 다시 불러오는 방법을 학습한다.

---

## 학습 목표

- 저장 계층의 종류(UserDefaults, SwiftData, 파일)를 이해한다.
- `UserDefaults`로 간단한 데이터를 저장한다.
- `SwiftData`(@Model, @Query, ModelContainer)로 구조화된 데이터 CRUD를 구현한다.
- 커스텀 타입을 JSON으로 직렬화해 저장한다.

---

## 1. 저장 방식 비교

| 방식 | 쓰임 | 특징 |
|------|------|------|
| UserDefaults | 설정·간단 값 | 키-값 저장, 대량 데이터에 비추천 |
| SwiftData | 논리적 데이터 모델 | 관계형, SwiftUI와 완벽 통합, iOS 17+ |
| 파일(JSON) | 백업/공유용 데이터 | 파일 1개로 저장 |
| SQLite | 대규모 | KMP 챕터에서 SQLDelight로 학습 |

---

## 2. UserDefaults

### 기본 사용

```swift
import Foundation

// 저장
UserDefaults.standard.set("Minho", forKey: "userName")
UserDefaults.standard.set(27, forKey: "userAge")
UserDefaults.standard.set(true, forKey: "isDarkMode")

// 읽기
let name = UserDefaults.standard.string(forKey: "userName") ?? "게스트"
let age = UserDefaults.standard.integer(forKey: "userAge")       // 없으면 0
let dark = UserDefaults.standard.bool(forKey: "isDarkMode")      // 없으면 false

print("\(name) / \(age) / \(dark)")
```

### 커스텀 구조체 저장 (JSON 직렬화)

```swift
struct Profile: Codable {
    var nickname: String
    var level: Int
}

let profile = Profile(nickname: "민호", level: 3)

// 저장
if let data = try? JSONEncoder().encode(profile) {
    UserDefaults.standard.set(data, forKey: "profile")
}

// 읽기
if let data = UserDefaults.standard.data(forKey: "profile"),
   let loaded = try? JSONDecoder().decode(Profile.self, from: data) {
    print(loaded.nickname)   // "민호"
}
```

> 유의: UserDefaults는 앱 삭제 시 사라지고, 보안/용량에 한계가 있습니다.

---

## 3. SwiftData

iOS 17+/macOS 14+ 에서 사용하는 영속화 프레임워크입니다. SwiftUI와 자연스럽게 연결됩니다.

### 모델 정의

```swift
import SwiftData
import Foundation

@Model
final class TaskItem {
    var title: String
    var note: String = ""
    var priority: Int = 1      // 1~3
    var isDone: Bool = false
    var createdAt: Date = Date()

    init(title: String, priority: Int = 1) {
        self.title = title
        self.priority = priority
    }
}
```

> `@Model` 매크로가 자동으로 저장/조회 능력을 부여합니다.

### Container/Context 설정

앱 진입점:

```swift
import SwiftUI

@main
struct SwiftDataTodoApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: TaskItem.self)   // 컨테이너 연결
    }
}
```

### 조회: @Query

```swift
struct ContentView: View {
    @Query private var tasks: [TaskItem]      // 자동 검색, 변경되면 갱신

    var body: some View {
        ...
    }
}
```

조건/정렬 쿼리:

```swift
@Query(sort: \TaskItem.createdAt, order: .reverse)
private var tasks: [TaskItem]

@Query(filter: #Predicate<TaskItem> { $0.isDone == true })
private var doneTasks: [TaskItem]
```

### 추가 / 삭제 / 수정

```swift
import SwiftData

struct TaskView: View {
    @Environment(\.modelContext) private var context
    @Query private var tasks: [TaskItem]
    @State private var newTitle = ""

    var body: some View {
        NavigationStack {
            List {
                ForEach(tasks) { task in
                    HStack {
                        Image(systemName: task.isDone ? "checkmark.circle.fill" : "circle")
                        Text(task.title)
                            .strikethrough(task.isDone)
                            .foregroundStyle(task.isDone ? .secondary : .primary)
                    }
                    .onTapGesture { toggle(task) }
                }
                .onDelete(perform: deleteTasks)
            }
            .navigationTitle("할 일")
            .safeAreaInset(edge: .bottom) {
                HStack {
                    TextField("새 할 일", text: $newTitle)
                        .textFieldStyle(.roundedBorder)
                    Button("추가") { addTask() }
                        .buttonStyle(.borderedProminent)
                }
                .padding()
            }
        }
    }

    func addTask() {
        guard !newTitle.isEmpty else { return }
        context.insert(TaskItem(title: newTitle))   // 추가
        newTitle = ""
    }

    func toggle(_ task: TaskItem) {
        task.isDone.toggle()                        // 수정 (자동 저장)
    }

    func deleteTasks(_ indexSet: IndexSet) {
        for index in indexSet {
            context.delete(tasks[index])            // 삭제
        }
    }
}
```

### 자동 저장 확인

SwiftData는 컨텍스트 변경을 자동 추적/저장합니다. 앱을 종료했다가 다시 실행해도 목록이 남아 있는지 확인해 보세요.

---

## 4. 파일 저장 (JSON을 파일로)

백업/공유를 위해 JSON 파일로 저장할 때:

```swift
import Foundation

struct JSONFileStore {
    static let fileURL: URL = {
        let docs = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
        return docs.appendingPathComponent("data.json")
    }()

    static func save<T: Encodable>(_ value: T) throws {
        let data = try JSONEncoder().encode(value)
        try data.write(to: fileURL)
    }

    static func load<T: Decodable>(_ type: T.Type) throws -> T {
        let data = try Data(contentsOf: fileURL)
        return try JSONDecoder().decode(type, from: data)
    }
}

// 사용
struct Note: Codable, Identifiable {
    var id = UUID()
    var text: String
}

let notes = [Note(text: "메모1"), Note(text: "메모2")]
try? JSONFileStore.save(notes)

let loaded: [Note]? = try? JSONFileStore.load([Note].self)
```

---

## 5. 실전 예제: 메모 앱 (SwiftData)

`NotesApp` 프로젝트를 만들어 완성해 보세요.

**요구사항**

1. @Model `Note`: `title`, `content`, `createdAt`
2. `.modelContainer(for: Note.self)` 연결
3. 목록 화면: @Query 정렬(최신순) + 스와이프 삭제 + 탭하면 편집 화면
4. 편집 화면: TextField/TextEditor로 수정
5. "추가" 버튼: 새 Note insert 후 편집 화면으로 이동

**실습 힌트**

- 편집 화면에서 `@Bindable var note: Note`를 쓰면 TextField(`$note.title`)로 바로 수정 가능:

```swift
struct NoteEditView: View {
    @Bindable var note: Note

    var body: some View {
        VStack {
            TextField("제목", text: $note.title)
                .font(.title2)
            TextEditor(text: $note.content)
        }
        .padding()
        .navigationTitle("메모 편집")
    }
}
```

---

## 🛠 실습: 설정 저장 앱 (UserDefaults)

**요구사항**

1. 설정 화면: 닉네임 TextField, 다크 모드 Toggle, 음악 볼륨 Slider
2. 나가서 다시 들어와도 값 유지 (UserDefaults)
3. `.onAppear`에서 불러오기, `.onChange`에서 저장
4. 추가: 좋아하는 언어 목록을 `[String]`으로 UserDefaults 저장/로드

**실습 힌트**

```swift
@State private var nickname = ""
@State private var isDark = false
@State private var volume: Double = 0.5

.onAppear {
    nickname = UserDefaults.standard.string(forKey: "nickname") ?? ""
    isDark = UserDefaults.standard.bool(forKey: "isDark")
    volume = UserDefaults.standard.double(forKey: "volume")
}
.onChange(of: volume) {
    UserDefaults.standard.set(volume, forKey: "volume")
}
```

---

## 마무리 & 다음 챕터

UserDefaults(짧은 설정)와 SwiftData(구조적 데이터) 저장을 구현했습니다.

다음 챕터 `06_Combine과_심화기능.md`에서 Combine과 애니메이션 등 iOS 앱을 더 완성도 있게 만드는 심화 기능을 학습합니다.