# app / android

> **트리거**: `androidapp/` 하위 파일을 생성·수정할 때 이 문서를 참조한다.

## 역할
- Android 애플리케이션 모듈. 런처 Activity, AndroidManifest, 플랫폼 초기화를 담당한다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `androidapp/src/main/AndroidManifest.xml` | 매니페스트 |
| `androidapp/src/main/kotlin/{package}/MainActivity.kt` | 단일 Activity (Compose 호스트) |
| `androidapp/src/main/kotlin/{package}/MainApplication.kt` | Application 클래스, DI 초기화 |

## 의존/연동
- `shared` 및 `core:*` 모듈을 사용한다.
- WorkManager 등 Android 전용 백그라운드 작업을 포함할 수 있다.

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| 새 Activity 추가 | 단일 Activity 원칙을 유지하고 있는지 먼저 확인. Navigation으로 대체 가능하면 추가하지 않는다 |
| 권한 추가 | `AndroidManifest.xml`에 추가 + 런타임 권한 요청 로직이 필요한지 판단 |
| 백그라운드 작업 추가 | WorkManager 사용. `shared` 모듈의 동기화 로직과 중복 여부 확인 |
| DI 초기화 변경 | `MainApplication`에서 `shared` 모듈의 DI 초기화 순서 확인 |

## DO / DON'T

```kotlin
// DO — shared 모듈을 통해 초기화
class MainApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        initKoin(androidContext = this)   // shared의 DI 초기화
    }
}

// DON'T — androidapp에서 직접 Repository/DataSource 생성
class MainApplication : Application() {
    val repo = ItemRepositoryImpl(...)   // 금지: DI를 우회
}
```

## 변경 시 체크리스트
- [ ] 앱 초기화 순서가 `shared` 모듈과 정합한가?
- [ ] 매니페스트 권한/서비스 등록이 런타임 동작과 일치하는가?
- [ ] 백그라운드 작업 변경이 동기화 주기에 영향을 주지 않는가?
- [ ] Gradle 의존성 변경 시 `shared` 모듈과 버전 충돌이 없는가?

## 빌드/실행
- `./gradlew :androidapp:installDebug` 로 실행한다.
