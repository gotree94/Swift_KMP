# 1.1 첫 Swift 프로그래밍과 기초 문법

> Swift 언어의 뼈대인 변수·상수·자료형·문자열을 익히고, 터미널에서 첫 프로그램을 실행한다.

---

## 학습 목표

- Swift 파일을 만들고 `swift` 명령으로 실행할 수 있다.
- `let`(상수)과 `var`(변수)의 차이를 이해한다.
- Int, Double, String, Bool 자료형과 타입 추론을 이해한다.
- 문자열 보간법으로 출력을 자유자재로 할 수 있다.

---

## 1. 첫 프로그램: Hello, Swift!

폴더를 만들고 파일을 생성합니다.

```bash
cd ~/Desktop/workspace
mkdir swift-basics
cd swift-basics
touch hello.swift
```

`hello.swift`를 Xcode나 텍스트 에디터로 열어 아래를 작성합니다.

```swift
import Foundation

print("Hello, Swift!")
```

실행:

```bash
swift hello.swift
```

출력 결과:

```
Hello, Swift!
```

> `import Foundation`은 기본 표준 라이브러리 중 Foundation 유틸리티(문자열 처리 등)를 불러옵니다.
> 앞으로 모든 실습은 `swift 파일명.swift`로 실행합니다.

---

## 2. 상수와 변수

Swift는 `let`(상수: 값을 바꿀 수 없음)과 `var`(변수: 값을 바꿀 수 있음)로 나뉩니다.

```swift
var name = "Kim"        // 변수: 변경 가능
name = "Lee"            // ✅ 변경 가능

let birthYear = 1995    // 상수: 변경 불가
// birthYear = 1996     // ❌ 컴파일 에러!

print(name)
print(birthYear)
```

**왜 상수를 기본으로 쓸까?** 값이 바뀌지 않는다는 보장이 생겨 코드가 안전해집니다.
Apple 가이드라인도 "변수보다 상수를 우선 사용" 하라고 권장합니다.

- 재할당이 필요한 상황에만 `var`를 쓰세요.

---

## 3. 기본 자료형

```swift
let age: Int = 30                // 정수
let temperature: Double = 36.5   // 실수 (Float보다 Double 권장)
let letter: Character = "A"      // 한 글자 문자
let greeting: String = "Hello"   // 문자열
let isStudent: Bool = true       // 불리언

print(type(of: age))             // "Int" 출력 (타입 확인)
```

### 타입 추론 (Type Inference)

타입을 명시하지 않아도 초기값으로 Swift가 타입을 추론합니다.

```swift
let count = 10        // Int로 추론
let pi = 3.14         // Double로 추론
let yes = false       // Bool로 추론
let msg = "Hi"        // String으로 추론
```

> 메모리 크기: Int(64bit), Double(64bit 부동소수), String(가변 길이)
> 정수·실수 혼합 연산 시에는 명시적 변환이 필요합니다:

```swift
let a: Int = 10
let b: Double = 2.5
// let c = a + b      // ❌ 타입 불일치 에러
let c = Double(a) + b // ✅ 12.5
```

---

## 4. 문자열

### 문자열 보간법 (String Interpolation)

`\(변수명)` 형태로 문자열 안에 값을 넣습니다.

```swift
let name = "Minho"
let age = 27
let message = "이름은 \(name)이고, \(age)살입니다."
print(message)   // 이름은 Minho이고, 27살입니다.

// 연산도 가능
let total = "결과: \(10 + 5)"   // "결과: 15"
```

### 문자열 결합

```swift
let first = "Swift"
let second = "Study"
let joined = first + " " + second   // "Swift Study"
var word = "Hello"
word += " World"                    // "Hello World" (var만 가능)
print(joined)
print(word)
```

### 문자열 주요 기능

```swift
let sentence = "I love Swift"

print(sentence.count)          // 글자 수: 12
print(sentence.uppercased())   // "I LOVE SWIFT"
print(sentence.lowercased())   // "i love swift"
print(sentence.hasPrefix("I"))  // true
print(sentence.contains("Sw"))  // true

for char in sentence {
    print(char)                // 한 글자씩 출력
}
```

---

## 5. 주석

```swift
// 한 줄 주석

/*
 여러 줄 주석
 /* 중첩 주석도 허용 */
*/
```

---

## 6. print와 debugPrint

```swift
print("기본 출력")            // 값 자체를 출력
debugPrint("디버그 출력")      // 표현식까지 상세 출력 (따옴표 포함)
```

---

## 🛠 실습 1: 본인 소개 프로그램

`intro.swift` 파일을 만들어 다음 요구사항을 만족하세요.

```bash
touch intro.swift
```

**요구사항**

1. 상수 `name`, `age`, `favoriteLanguage`를 선언합니다.
2. 상수 `height`을 실수형으로 선언합니다.
3. 모든 값을 문자열 보간법으로 한 문장에 출력합니다.
4. 실행 결과 예시:

```
제 이름은 Minho이고, 27살입니다. 키는 175.5cm이며, 좋아하는 언어는 Swift입니다.
```

**실습 힌트**

- `let height: Double = 175.5` 로 선언하면 됩니다.
- 한 print문에 다 넣으면 됩니다.

---

## 🛠 실습 2: 자판기 프로그램 (입출력 연습)

이번에는 사용자 입력을 받아보겠습니다.

```swift
import Foundation

print("음료 가격을 입력하세요:", terminator: " ")
if let input = readLine(), let price = Int(input) {
    let tax = price * 10 / 100
    print("가격: \(price)원, 부가세: \(tax)원, 총액: \(price + tax)원")
} else {
    print("잘못된 입력입니다.")
}
```

- 실행 → `1234` 입력 → `가격: 1234원, 부가세: 123원, 총액: 1357원`
- `terminator: " "`은 print 줄바꿈을 막아 같은 줄에 입력받게 합니다.
- `Int(input)`은 문자열을 정수로 변환하는 옵셔널 결과입니다.

**변형 과제**: 메뉴 3개의 가격을 상수로 미리 정의하고, 선택한 메뉴번호에 따라 가격과 총액을 출력하세요.

---

## 🛠 실습 3: 조건 없는 계산기 (자료형 변환)

```swift
import Foundation

let a: Double = 9
let b: Double = 4

print("\(a) + \(b) = \(a + b)")
print("\(a) - \(b) = \(a - b)")
print("\(a) * \(b) = \(a * b)")
print("\(a) / \(b) = \(a / b)")    // 2.25
print("\(a)의 \(b)제곱 = \(pow(a, b))")  // 6561.0
```

**변형 과제**: 위 코드를 `let x: Int = 9, let y: Int = 4`로 바꾸고, 나눗셈이 `2.25`가 되도록 Double 변환을 적용해 개선하세요.

---

## 실습 풀이

<details>
<summary>실습 1 정답 확인</summary>

```swift
let name = "Minho"
let age = 27
let favoriteLanguage = "Swift"
let height: Double = 175.5

print("제 이름은 \(name)이고, \(age)살입니다. 키는 \(height)cm이며, 좋아하는 언어는 \(favoriteLanguage)입니다.")
```
</details>

<details>
<summary>실습 2 정답 확인</summary>

```swift
import Foundation

let menu = [0, 1200, 1800, 2500]  // 0은 더미
print("1. 아메리카노 1200원")
print("2. 라떼 1800원")
print("3. 스무디 2500원")

print("선택:", terminator: " ")
if let input = readLine(), let choice = Int(input), choice >= 1, choice <= 3 {
    let price = menu[choice]
    let tax = price * 10 / 100
    print("가격: \(price)원, 부가세: \(tax)원, 총액: \(price + tax)원")
} else {
    print("잘못된 입력입니다.")
}
```
</details>

---

## 다음 챕터로

다음은 `02_데이터타입과_컬렉션.md`에서 배열·딕셔너리 등 여러 값을 담는 자료구조를 학습합니다.