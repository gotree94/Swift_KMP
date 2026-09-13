# 6.3 CI/CD와 배포

> GitHub Actions로 KMP 프로젝트를 자동 빌드·테스트하고, iOS/Android 스토어에 배포하는 절차를 학습한다.

---

## 학습 목표

- CI(지속적 통합)의 개념과 워크플로우를 이해한다.
- GitHub Actions로 KMP 프로젝트를 자동 빌드·테스트한다.
- Android 앱 APK/AAB 빌드와 배포 방식을 안다.
- iOS 앱 아카이브와 앱스토어 배포(Tesflight/App Store) 흐름을 안다.

---

## 1. CI란 무엇인가?

- **Continuous Integration**: 코드가 원격 저장소(GitHub)에 올라갈 때마다 자동으로 빌드·테스트 실행
- 개발자는 merge 전에 실패를 미리 캐치

```
Push → GitHub Actions → build → test → (PR 상태 등록)
```

KMP는 3가지를 확인하는 것이 일반적입니다:
1. common 코드가 양 플랫폼에서 컴파일되는지
2. 공용/Android 테스트가 통과하는지
3. iOS 프레임워크가 빌드되는지

---

## 2. GitHub Actions 워크플로우 기본

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build-and-test:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      # Gradle 캐시
      - uses: gradle/gradle-build-action@v3

      # Android 라이브러리 + 공용 테스트
      - run: ./gradlew :shared:assembleDebug :shared:testDebugUnitTest

      # iOS 프레임워크 컴파일 확인
      - run: ./gradlew :shared:linkDebugFrameworkIosSimulatorArm64

      # (선택) iOS 공용 테스트
      - run: ./gradlew :shared:iosSimulatorArm64Test
```

> ios test가 느리면 `linkDebugFrameworkIosSimulatorArm64`만 컴파일 확인으로 두기도 합니다.

### 실행 트리거 및 분기

| 이벤트 | 의미 |
|--------|------|
| `push: branches: [main]` | main에 push 시 |
| `pull_request:` | PR 만들 때 |
| `schedule:` | 정기 실행 |
| `workflow_dispatch:` | 수동 실행 (버튼) |

---

## 3. 캐시 & 버전 관리

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
```

- 첫 빌드는 의존성 다운로드로 오래 걸리고, 이후 캐시로 빨라집니다.
- macOS 러너는 유료 구독에서 병렬 사용 가능(기본 포함 범위와 확인 필요). 프리 티어 macOS 러너는 제한이 있을 수 있으므로 설정 확인.

---

## 4. Android 배포 (APK/AAB)

### Build

```bash
./gradlew :androidApp:assembleDebug                    # 디버그 APK
./gradlew :androidApp:assembleRelease                  # 릴리스 APK (서명 필요)
./gradlew :androidApp:bundleRelease                    # AAB (Play 스토어용)
```

출력물:
- APK: `androidApp/build/outputs/apk/...`
- AAB: `androidApp/build/outputs/bundle/...`

### 서명 설정 (release)

`androidApp/build.gradle.kts`:

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile   = file("../keystore.jks")
            storePassword   = System.getenv("KEYSTORE_PASSWORD")
            keyAlias        = System.getenv("KEYSTORE_ALIAS")
            keyPassword     = System.getenv("KEY_PASSWORD")
        }
    }
}
```

> 비밀키는 git에 절대 커밋하지 않습니다. GitHub Secret에 저장합니다.

### Play Console 배포

1. Google Play Console → 앱 만들기
2. `Upload to Production` → `bundleRelease` AAB 업로드
3. 상세 정보/심사 제출

### GitHub Action 예 (Play Upload)

```yaml
- name: Build release AAB
  run: ./gradlew :androidApp:bundleRelease

- name: Upload to Play Console
  uses: r0adkll/upload-google-play@v1
  with:
    serviceAccountJsonPlainText: ${{ secrets.SERVICE_ACCOUNT_JSON }}
    packageName: com.example.myapp
    releaseFiles: androidApp/build/outputs/bundle/release/androidApp-release.aab
    track: internal
```

---

## 5. iOS 배포 (App Store / TestFlight)

### 아카이브 및 분배

Xcode에서:
1. `Product > Archive`
2. `Distribute App` → `App Store Connect`
3. Apple ID 로그인 → TestFlight 배포 선택

### CLI 아카이브 (CI 스크립트)

```bash
xcodebuild -workspace iosApp.xcworkspace \
  -scheme iosApp \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/iosApp.xcarchive \
  archive

xcodebuild -exportArchive \
  -archivePath build/iosApp.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath build/export
```

> `ExportOptions.plist`에 `method: app-store-connect` 지정

### 귀찮은 전제 조건

1. **Apple Developer Program** (유료 연회비 $99) 가입
2. App Store Connect에서 앱 이터너(스키마: bundle id) 생성, 팀 등록
3. 인증서(Certificate) + 프로비저닝 프로파일 설정
4. Fastlane이 자주 쓰입니다 (자동화 편의)

### Fastlane 기본

```ruby
# fastlane/Fastfile
lane :beta do
  match(type: "appstore", readonly: true)
  gym(scheme: "iosApp")                  # 빌드
  pilot(distribute_external: true)       # TestFlight 업로드
end
```

```bash
bundle exec fastlane beta
```

> Fastlane은 인증서 관리(match), 빌드(gym), 업로드(pilot)를 자동화합니다.
> KMP 템플릿에는 `iosApp` 스키마가 이미 있어 그대로 사용 가능합니다.

---

## 6. 예제 워크플로우 (KMP 전체 CI)

```yaml
name: KMP CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  unit-tests:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - uses: gradle/gradle-build-action@v3
      - run: ./gradlew :shared:testDebugUnitTest

  ios-framework:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - uses: gradle/gradle-build-action@v3
      - run: ./gradlew :shared:linkDebugFrameworkIosSimulatorArm64

  android-build:
    runs-on: ubuntu-latest        # Android 빌드는 ubuntu로 빨라짐
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '17' }
      - uses: gradle/gradle-build-action@v3
      - run: ./gradlew :androidApp:assembleDebug
```

---

## 🛠 실습 1: 기본 CI 구성

**요구사항**

1. 저장소에 `.github/workflows/ci.yml` 추가
2. Android 라이브러리 테스트 + iOS 프레임워크 컴파일 검사 워크플로우 작성
3. PR 생성 시 자동 실행되는지 확인
4. 의도적으로 실패 테스트(또는 잘못된 코드)를 커밋해 Action이 실패하는 모습 관찰 후 고치기

---

## 🛠 실습 2: 릴리스 빌드 파이프라인

**요구사항**

1. `workflow_dispatch`(수동 실행) 가능한 `release.yml` 작성
2. Android `bundleRelease` + iOS `archive`를 단계별로 실행
3. 아티팩트(APK, xcarchive zip)를 `actions/upload-artifact`로 저장
4. 키/인증서는 GitHub Secrets로만 참조

**실습 힌트**

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: android-release
    path: androidApp/build/outputs/bundle/release/*.aab
```

---

## 🛠 실습 3: (선택) 테스트 자동화 확장

- `kotlinx-coroutines-test`가 포함된 commonTest를 CI에서 실행
- iOS 시뮬레이터 테스트를 `xcrun simctl` 또는 `xcodebuild test`로 연결
- 시간이 허락하면 Fastlane pilot → TestFlight까지 시도

---

## 마무리 & 다음 챕터

CI/CD가 갖춰지면 merge 마다 빌드/테스트가 자동 검증됩니다.

마지막 챕터 `04_마스터_프로젝트.md`에서 지금까지 배운 모든 것을 종합한 프로젝트를 진행합니다.