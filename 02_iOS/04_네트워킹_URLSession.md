# 2.4 네트워킹: URLSession과 JSON

> 실제 서버와 HTTP 통신을 하고, JSON을 디코딩하여 화면에 뿌리는 방법을 학습한다.

---

## 학습 목표

- `URLSession`으로 GET/POST 요청을 보낼 수 있다.
- `async/await` 기반의 비동기 통신 코드를 작성할 수 있다.
- `Codable`로 JSON을 모델로 변환(디코딩)한다.
- 오류 상황(상태 코드, 네트워크 실패)을 처리한다.

---

## 1. HTTP 통신 기초

- 클라이언트(iOS 앱) → 서버: `https://api.example.com/users` 같은 URL에 요청
- 서버가 응답: 상태 코드(200 성공, 404 없음, 500 오류) + JSON

테스트용 공개 API를 활용합니다:
- https://dummyjson.com/products — 상품 목록
- https://jsonplaceholder.typicode.com/posts — 게시글

---

## 2. 신형 네트워킹: async/await

iOS 15+ 에서는 SwiftUI와 잘 맞는 `async/await` 문법을 씁니다.

### 모델 정의 (Codable)

```swift
struct Post: Codable, Identifiable {
    let id: Int
    let title: String
    let body: String
}
```

### 통신 함수

```swift
import Foundation

struct APIClient {
    enum APIError: Error {
        case invalidURL
        case badResponse(Int)
        case noData
    }

    static func fetchPosts() async throws -> [Post] {
        guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
            throw APIError.invalidURL
        }

        // 1) 요청
        let (data, response) = try await URLSession.shared.data(from: url)

        // 2) 상태 코드 검사
        guard let http = response as? HTTPURLResponse,
              (200..<300).contains(http.statusCode) else {
            throw APIError.badResponse((response as? HTTPURLResponse)?.statusCode ?? -1)
        }

        // 3) JSON 디코딩
        let posts = try JSONDecoder().decode([Post].self, from: data)
        return posts
    }
}
```

> `URLSession.shared.data(from:)` 는 내부적으로 비동기지만 `await`로 편하게 씁니다.
> 비동기는 화면 로딩을 막지 않습니다.

---

## 3. 뷰에서 사용하기 (async 클로저)

### .task로 화면 등장 시 로드

```swift
import SwiftUI

struct PostListView: View {
    @State private var posts: [Post] = []
    @State private var isLoading = false
    @State private var errorMessage: String?

    var body: some View {
        List {
            if isLoading {
                ProgressView("불러오는 중...")
            } else if let errorMessage {
                Text(errorMessage).foregroundStyle(.red)
            } else {
                ForEach(posts) { post in
                    VStack(alignment: .leading) {
                        Text(post.title)
                            .font(.headline)
                        Text(post.body)
                            .font(.caption)
                            .foregroundStyle(.secondary)
                            .lineLimit(2)
                    }
                }
            }
        }
        .navigationTitle("게시글")
        .task {
            await loadPosts()
        }
    }

    func loadPosts() async {
        isLoading = true
        errorMessage = nil
        do {
            posts = try await APIClient.fetchPosts()
        } catch {
            errorMessage = "불러오기 실패: \(error.localizedDescription)"
        }
        isLoading = false
    }
}
```

### 실패 시 재시도 버튼

```swift
} else if let errorMessage {
    VStack {
        Text(errorMessage)
        Button("다시 시도") {
            Task { await loadPosts() }
        }
    }
}
```

---

## 4. POST 요청 (데이터 보내기)

```swift
struct NewPostRequest: Encodable {
    let userId: Int
    let title: String
    let body: String
}

func createPost(title: String) async throws -> Post {
    guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts") else {
        throw APIError.invalidURL
    }

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let body = NewPostRequest(userId: 1, title: title, body: "내용")
    request.httpBody = try JSONEncoder().encode(body)

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let http = response as? HTTPURLResponse,
          (200..<300).contains(http.statusCode) else {
        throw APIError.badResponse((response as? HTTPURLResponse)?.statusCode ?? -1)
    }

    return try JSONDecoder().decode(Post.self, from: data)
}
```

---

## 5. 이미지 로딩 (비동기 이미지)

```swift
// placeholder: 로딩 중 표시될 기본 이미지
AsyncImage(url: URL(string: "https://dummyjson.com/image/300x300")) { phase in
    switch phase {
    case .success(let image):
        image.resizable().scaledToFit()
    case .failure:
        Image(systemName: "wifi.exclamationmark")
            .foregroundStyle(.red)
    default:
        ProgressView()
    }
}
.frame(width: 200, height: 200)
```

---

## 6. 실전 예제: 상품 목록 앱

`ProductApp` 프로젝트를 만들고 dummyjson API로 상품 앱을 만들어 봅시다.

### 모델

```swift
struct Product: Codable, Identifiable {
    let id: Int
    let title: String
    let price: Double
    let description: String
    let thumbnail: String
}

struct ProductResponse: Codable {
    let products: [Product]
}
```

### API 함수

```swift
struct ProductAPI {
    static func fetchProducts() async throws -> [Product] {
        let url = URL(string: "https://dummyjson.com/products")!
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
            throw URLError(.badServerResponse)
        }
        let decoded = try JSONDecoder().decode(ProductResponse.self, from: data)
        return decoded.products
    }
}
```

### 리스트 화면

```swift
struct ProductListView: View {
    @State private var products: [Product] = []
    @State private var isLoading = true
    @State private var errorMessage: String?

    var body: some View {
        NavigationStack {
            List(products) { product in
                NavigationLink {
                    ProductDetailView(product: product)
                } label: {
                    HStack {
                        AsyncImage(url: URL(string: product.thumbnail))
                            .frame(width: 60, height: 60)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                        VStack(alignment: .leading) {
                            Text(product.title).font(.headline)
                            Text("$\(product.price, specifier: "%.0f")")
                                .foregroundStyle(.blue)
                        }
                    }
                }
            }
            .navigationTitle("상품 목록")
            .overlay {
                if isLoading { ProgressView() }
                else if let errorMessage {
                    VStack(spacing: 12) {
                        Text(errorMessage).foregroundStyle(.red)
                        Button("다시 시도") { Task { await load() } }
                    }
                }
            }
            .task { await load() }
        }
    }

    func load() async {
        isLoading = true
        errorMessage = nil
        do {
            products = try await ProductAPI.fetchProducts()
        } catch {
            errorMessage = error.localizedDescription
        }
        isLoading = false
    }
}
```

---

## 🛠 실습: 날씨 앱 만들기 (공개 API)

이번 실습은 반드시 손으로 작성하세요.

**요구사항**

1. **일일 365회 무료**인 openweathermap.org, 또는 아래 무료 날씨 API를 사용합니다.
   - 예: `https://api.open-meteo.com/v1/forecast?latitude=37.57&longitude=126.98&current_weather=true`
2. 모델 `Weather`: `temperature`, `windspeed`, `weathercode` 필드 사용
3. 화면: "서울 날씨" + 온도 + 풍속 + 상태 아이콘
4. `.task`로 로드, 실패 시 에러 메시지 + 재시도 버튼

**실습 힌트**

- 응답 JSON에서 `current_weather` 객체 안에 값이 있습니다:

```json
{
  "current_weather": {
    "temperature": 22.4,
    "windspeed": 11.2,
    "weathercode": 2
  }
}
```

- nil 코딩키가 필요합니다:

```swift
struct WeatherResponse: Codable {
    let currentWeather: CurrentWeather

    enum CodingKeys: String, CodingKey {
        case currentWeather = "current_weather"
    }
}
```

- 날씨 코드: `0`맑음, `1-3`구름, `45-48`안개, `51-67`비, `71-86`눈, `95+`뇌우

---

## 실습 풀이

<details>
<summary>날씨 앱 정답</summary>

```swift
import SwiftUI

struct CurrentWeather: Codable {
    let temperature: Double
    let windspeed: Double
    let weathercode: Int
}

struct WeatherResponse: Codable {
    let currentWeather: CurrentWeather

    enum CodingKeys: String, CodingKey {
        case currentWeather = "current_weather"
    }
}

struct WeatherAPI {
    static func fetchSeoul() async throws -> CurrentWeather {
        let url = URL(string: "https://api.open-meteo.com/v1/forecast?latitude=37.57&longitude=126.98&current_weather=true")!
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
            throw URLError(.badServerResponse)
        }
        let decoded = try JSONDecoder().decode(WeatherResponse.self, from: data)
        return decoded.currentWeather
    }
}

struct WeatherView: View {
    @State private var weather: CurrentWeather?
    @State private var errorMessage: String?

    func weatherText(_ code: Int) -> String {
        switch code {
        case 0: return "맑음 ☀️"
        case 1...3: return "구름 ⛅"
        case 45...48: return "안개 🌫️"
        case 51...67: return "비 🌧️"
        case 71...86: return "눈 ❄️"
        default: return "뇌우 ⛈️"
        }
    }

    var body: some View {
        VStack(spacing: 20) {
            if let weather {
                Text("서울")
                    .font(.largeTitle).fontWeight(.bold)
                Text("\(weather.temperature, specifier: "%.1f")°C")
                    .font(.system(size: 72))
                Text(weatherText(weather.weathercode))
                Text("풍속 \(weather.windspeed, specifier: "%.1f") km/h")
                    .foregroundStyle(.secondary)
            } else if let errorMessage {
                Text(errorMessage).foregroundStyle(.red)
                Button("다시 시도") { Task { await load() } }
            } else {
                ProgressView()
            }
        }
        .padding()
        .task { await load() }
    }

    func load() async {
        errorMessage = nil
        do {
            weather = try await WeatherAPI.fetchSeoul()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```
</details>

---

## 마무리 & 다음 챕터

URLSession + Codable + async/await 조합으로 서버 데이터를 화면에 표시했습니다.

다음 챕터 `05_데이터저장_SwiftData.md`에서 폐쇄 앱을 위해 데이터를 저장하고 다시 불러오는 방법을 학습합니다.