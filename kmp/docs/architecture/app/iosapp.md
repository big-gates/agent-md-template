# app / ios

> **트리거**: `iosApp/` 하위 파일을 생성·수정할 때 이 문서를 참조한다.

## 역할
- iOS 애플리케이션 모듈. SwiftUI 엔트리, AppDelegate, iOS 플랫폼 초기화를 담당한다.

## 표준 경로

| 경로 | 설명 |
|------|------|
| `iosApp/iosApp/iOSApp.swift` | @main SwiftUI 엔트리 |
| `iosApp/iosApp/AppDelegate.swift` | UIApplicationDelegate (필요 시) |
| `iosApp/iosApp/Info.plist` | 앱 설정 |
| `iosApp/Podfile` 또는 `Package.swift` | 의존성 관리 |

## 의존/연동
- KMP `shared` 모듈과 브릿지한다.
- BGTask, 푸시 알림 등 iOS 전용 기능을 포함할 수 있다.

## 에이전트 판단 규칙

| 상황 | 판단 |
|------|------|
| Swift에서 Kotlin 호출 | `shared` 모듈의 public API를 통해서만 접근. 내부 `core:*`를 직접 호출하지 않는다 |
| 백그라운드 작업 추가 | `BGTaskScheduler` 사용. `shared` 모듈의 동기화 로직 호출 여부 확인 |
| 푸시 알림 변경 | `UNUserNotificationCenter` 권한 + `shared` 모듈의 토큰 등록 연동 확인 |
| CocoaPods/SPM 변경 | CI 빌드 확인 필요. Podfile.lock 또는 Package.resolved 갱신 |

## DO / DON'T

```swift
// DO — shared 모듈을 통해 초기화
@main
struct iOSApp: App {
    init() {
        KoinInitializer.shared.initialize()   // shared의 DI 초기화
    }
}

// DON'T — iOS에서 직접 비즈니스 로직 구현
struct iOSApp: App {
    func syncData() {
        URLSession.shared.dataTask(...)   // 금지: shared 모듈 우회
    }
}
```

## 변경 시 체크리스트
- [ ] `shared` 모듈의 초기화 호출 시점과 순서가 올바른가?
- [ ] SwiftUI 생명주기 이벤트에서 동기화 호출 타이밍이 적절한가?
- [ ] 백그라운드/푸시 변경이 동기화 동작에 영향을 주지 않는가?
- [ ] CocoaPods/SPM 의존성 변경 시 CI에서 빌드 성공하는가?

## 빌드/실행
- Xcode에서 실행하거나 `xcodebuild` CLI를 사용한다.
