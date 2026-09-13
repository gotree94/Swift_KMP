# Swift & Kotlin Multiplatform (KMP) 학습 커리큘럼

> Mac에서 iOS 개발과 Kotlin Multiplatform을 **기본기부터 고급**까지 체계적으로 배우는 로드맵입니다.
> 각 챕터는 개별 Markdown 파일로 구성되어 있으며, 이론 → 코드 예제 → 실습 순으로 진행됩니다.

---

## 🎯 목표

- **Swift/SwiftUI** 를 이용해 네이티브 iOS 앱을 만들 수 있다.
- **Kotlin/Jetpack Compose** 로 Android 앱의 기본기를 다진다.
- **Kotlin Multiplatform** 으로 iOS/Android 양쪽에서 재사용되는 공용 로직(네트워킹, 데이터 저장, 비즈니스 로직)을 구현할 수 있다.
- **Compose Multiplatform** 으로 두 플랫폼에서 공유하는 UI까지 작성할 수 있다.

---

## 📋 커리큘럼 전체 구조

### 0단계. 시작 전 준비
| 챕터 | 파일 | 내용 |
|------|------|------|
| 0.0 | `00_환경설정.md` | Mac 개발 환경 구축 (Xcode, Android Studio, JDK, Git) |

### 1단계. Swift 언어 마스터 (기초 ~ 중급)
| 챕터 | 파일 | 내용 |
|------|------|------|
| 1.1 | `01_Swift/01_첫_Swift와_기초_문법.md` | 첫 Swift 프로그램, 변수/상수, 자료형, 출력 |
| 1.2 | `01_Swift/02_데이터타입과_컬렉션.md` | 문자열, 배열, 딕셔너리, 집합, 튜플 |
| 1.3 | `01_Swift/03_제어흐름과_함수.md` | 조건문, 반복문, 함수, 가드문 |
| 1.4 | `01_Swift/04_클로저와_고차함수.md` | 클로저, map/filter/reduce |
| 1.5 | `01_Swift/05_구조체_클래스_열거형_프로토콜.md` | OOP 기초, 프로토콜, 확장 |
| 1.6 | `01_Swift/06_옵셔널_오류처리_제네릭.md` | 옵셔널 바인딩, 오류 처리, 제네릭 |

### 2단계. iOS 앱 개발 (SwiftUI)
| 챕터 | 파일 | 내용 |
|------|------|------|
| 2.1 | `02_iOS/01_SwiftUI_입문_첫앱.md` | 첫 SwiftUI 앱, View/Preview, 기본 컴포넌트 |
| 2.2 | `02_iOS/02_레이아웃과_데이터_흐름.md` | 스택 레이아웃, @State/@Binding/@Observable |
| 2.3 | `02_iOS/03_네비게이션과_리스트.md` | NavigationStack, List, 동적 데이터 |
| 2.4 | `02_iOS/04_네트워킹_URLSession.md` | HTTP 통신, JSON 디코딩, 비동기 |
| 2.5 | `02_iOS/05_데이터저장_SwiftData.md` | UserDefaults, SwiftData, 앱 영속화 |
| 2.6 | `02_iOS/06_Combine과_심화기능.md` | Combine, 이미지 캐시, 애니메이션 | 

### 3단계. Kotlin 언어 마스터
| 챕터 | 파일 | 내용 |
|------|------|------|
| 3.1 | `03_Kotlin/01_기초_문법.md` | Kotlin 기초, var/val, 함수, 컬렉션 |
| 3.2 | `03_Kotlin/02_객체지향과_함수형.md` | 클래스, 상속, 람다, 확장 함수, data class |
| 3.3 | `03_Kotlin/03_코루틴과_Flow.md` | 코루틴, suspend, Flow, 비동기 처리 |

### 4단계. Android 기초
| 챕터 | 파일 | 내용 |
|------|------|------|
| 4.1 | `04_Android/01_JetpackCompose_입문.md` | 첫 Android 앱, Compose UI, 상태 |

### 5단계. Kotlin Multiplatform (KMP)
| 챕터 | 파일 | 내용 |
|------|------|------|
| 5.1 | `05_KMP/01_KMP_개념과_프로젝트_구조.md` | KMP 동작 원리, 프로젝트 생성 |
| 5.2 | `05_KMP/02_expect_actual과_플랫폼_통합.md` | expect/actual, iOS에서 공용 코드 호출 |
| 5.3 | `05_KMP/03_공통_네트워킹_Ktor.md` | Ktor Client, 직렬화, 공용 API 계층 |
| 5.4 | `05_KMP/04_공통_데이터_저장_SQLDelight.md` | SQLDelight, 멀티플랫폼 설정 |
| 5.5 | `05_KMP/05_Compose_Multiplatform.md` | UI까지 공유하는 Compose MP |

### 6단계. 고급 심화
| 챕터 | 파일 | 내용 |
|------|------|------|
| 6.1 | `06_고급/01_아키텍처_패턴.md` | MVVM, Repository, DI, Clean Architecture |
| 6.2 | `06_고급/02_테스팅_전략.md` | 공용 로직 단위 테스트, iOS/Android 테스트 |
| 6.3 | `06_고급/03_CI_CD와_배포.md` | GitHub Actions, 앱 스토어 배포 |
| 6.4 | `06_고급/04_마스터_프로젝트.md` | 전체 기술을 활용한 완성형 앱 만들기 |

---

## 📅 추천 학습 일정 (주 5시간 기준)

| 주차 | 범위 | 예상 시간 |
|------|------|----------|
| 1주차 | 0단계 + 1.1~1.2 | 5시간 |
| 2주차 | 1.3~1.6 | 5시간 |
| 3~4주차 | 2.1~2.6 (iOS 앱) | 10시간 |
| 5~6주차 | 3.1~3.3 + 4.1 | 10시간 |
| 7~8주차 | 5.1~5.5 (KMP) | 10시간 |
| 9~10주차 | 6.1~6.4 (고급) | 10시간 |

---

## ✅ 학습 방식 가이드

1. 한 챕터의 **개념**을 먼저 읽고, 코드 예제를 손으로 따라 칩니다.
2. 챕터 끝의 **실습** 을 반드시 스스로 완성합니다 (정답은 바로 보지 않기).
3. 완성한 프로젝트는 GitHub에 커밋합니다.
4. 실습 중 막히면 챕터의 "실습 힌트" 를 참고합니다.
5. 각 단계가 끝나면 다음 단계로 넘어갑니다.

---

## 🧑‍💻 필요 요구사항

- Apple Silicon 또는 Intel Mac (macOS 14+ 권장)
- 최소 여유 공간 30GB (Xcode + Android Studio)
- 인터넷 연결 (패키지 다운로드 필요)

---

## 📁 생성할 폴더

```
Swift_KMP_커리큘럼/
├── README.md
├── 00_환경설정.md
├── 01_Swift/
├── 02_iOS/
├── 03_Kotlin/
├── 04_Android/
├── 05_KMP/
└── 06_고급/
```

> ⚠️ **중요**: 이 커리큘럼은 Mac 환경을 기준으로 작성되었습니다.
> Swift/SwiftUI/iOS를 학습하려면 반드시 Mac이 필요합니다.
> CLI 상에서 Swift 문법만 연습해보려면 Mac 없이 Linux/Windows에서도 Swift Toolchain 설치가 가능하나, iOS 시뮬레이터·Xcode 실습은 Mac에서만 가능합니다.