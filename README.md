# 차량운행일지 (VehicleLog)

안드로이드(삼성 갤럭시) **차량 운행일지 자동 생성** 앱.
차량 블루투스 연결로 운행을 자동 감지·기록하고, 등록 장소 기반으로 업무/출퇴근/개인 분류,
한솔아이원스 사업장 간 고정거리 적용, **여비교통비(국내) 청구 엑셀**로 내보내기. **인앱 자동 업데이트** 지원.

> 이 저장소(`vehiclelog-dist`)는 **APK 배포 채널 + 프로젝트 문서**입니다.
> **소스코드는 개발 PC의 `D:\VehicleLog`** 에 있습니다(이 저장소엔 README와 APK 릴리스만).

설치(사용자): [최신 릴리스](https://github.com/junjang41-boop/vehiclelog-dist/releases/latest) APK 다운로드 → "이 출처 앱 설치 허용" → 설치. 안드로이드 8.0+.

---

## 동작 원리
- 상시 **포그라운드 서비스**(`TripTrackingService`)가 동적 BroadcastReceiver로 차량 BT의 `ACL_CONNECTED/DISCONNECTED` 감지.
- 연결 = 운행 시작 / 해제 = 종료(**5분 디바운스** 병합). 앱 켤 때 이미 연결돼 있으면 `checkAlreadyConnected()`(A2DP/HEADSET 프로파일)로 즉시 시작.
- 운행 중 FusedLocation으로 GPS 점(`trip_points`) 기록 → 거리 계산 + 경로 지도.
- 종료 시: 출발/도착이 등록 장소면 **업무/출퇴근/개인 자동 분류**, 사업장↔사업장이면 **회사 지정 고정거리(편도)** 사용, 역지오코딩 주소, **'업무목적 작성' 알림**(미확정 시 30분 주기 WorkManager 반복).
- 내보내기: 업무 운행을 회사 양식 **C~K열 순서 .xlsx**로 생성 → 양식 C4에 붙여넣기(유류대/감가상각/합계 수식은 양식이 자동 계산).

## 기술 스택
- Kotlin 1.9.24 · AGP 8.5.2 · Gradle 8.9 · JDK 17
- Jetpack Compose (BOM 2024.06) + Material3 · Navigation Compose
- Room 2.6.1 (KSP) · DataStore Preferences
- play-services-location · accompanist-permissions · osmdroid 6.1.18 · WorkManager 2.9
- 패키지 `com.vehiclelog.app` · minSdk 26 · target/compile 34 · **디버그 서명(사이드로드)**

## 빌드 (개발 PC: Windows)
한글 경로에서 안드로이드 빌드가 깨져 **ASCII 경로 `D:\VehicleLog`** 사용. 최종 APK는 `D:\차량일지자동생성`(사용자 폴더)로 복사.

툴체인(이 PC에 직접 설치):
- JDK 17: `D:\VehicleLog\.toolchain\jdk\jdk-17.0.19+10`
- Android SDK: `D:\VehicleLog\.toolchain\android-sdk` (platform-tools, platforms;android-34, build-tools;34.0.0)
- Gradle wrapper 8.9 · gh CLI: `D:\VehicleLog\.toolchain\gh\bin\gh.exe`

```powershell
$env:JAVA_HOME="D:\VehicleLog\.toolchain\jdk\jdk-17.0.19+10"
& "D:\VehicleLog\gradlew.bat" -p "D:\VehicleLog" :app:assembleDebug
# 결과: app/build/outputs/apk/debug/app-debug.apk
```

## 배포 (인앱 자동 업데이트)
- 앱의 `util/UpdateChecker.REPO = "junjang41-boop/vehiclelog-dist"` 가 `releases/latest` 조회 → 태그 `vN`(N=versionCode)이 설치된 versionCode보다 크면 업데이트 다이얼로그.
- 릴리스: `app/build.gradle.kts`의 `versionCode`/`versionName`을 올린 뒤
  ```powershell
  powershell -ExecutionPolicy Bypass -File D:\VehicleLog\release.ps1 -Notes "변경점"
  ```
  (빌드 + APK 복사 + `gh release create vN`)
- ⚠️ **반드시 같은 PC(같은 디버그 키스토어)로 빌드**해야 인플레이스 업데이트가 됨.
- 무탭 완전자동은 Play 스토어만 가능. 사이드로드는 "다운로드 + 설치 1탭".

## 프로젝트 구조 (`app/src/main/java/com/vehiclelog/app`)
- `data/` Room(Entity: Vehicle/Place/Trip/TripPoint, DAO, AppDatabase **v3**), Repository, SettingsStore(DataStore), Enums(PlaceCategory/TripPurpose/FuelType), **CompanySites**, **SiteSeeder**
- `service/` **TripTrackingService**(핵심 감지/기록), MonitoringController, ReminderWorker(WorkManager)
- `receiver/` BootReceiver, TripActionReceiver(알림 액션)
- `ui/` nav/AppNav · screen/(Home, TripList, TripDetail, Vehicle, PlaceList, PlaceEdit, Settings, Export) · vm/ · component/ · theme/
- `util/` GeoUtil(거리), GeocoderHelper(정/역 지오코딩), TripClassifier, TimeUtil, NotificationHelper, TripReminder, **MinimalXlsx**(의존성 없는 xlsx 생성), **UpdateChecker**

## 회사 특화 데이터 (`data/CompanySites.kt`)
- 사업장 4곳 + 안성2 주차장. siteKey: `HQ` / `ANSEONG2` / `RND` / `BALAN`. 첫 실행 시 `SiteSeeder`가 자동 등록(`seedVersion`로 업데이트 시 신규 시드 추가, 이름 기준 dedup).
- 고정 편도거리(km): 본사–발안 48 · 본사–기술 25 · 본사–안성2 16 · 발안–기술 29 · 발안–안성2 45 · 기술–안성2 35.
- 발안 좌표는 향남읍 근사값 → 기기 지오코딩/지도 핀으로 보정 필요. 도착 인식 반경 기본 150m.
- 여비교통비 양식 데이터칸 **C~K**: 날짜·요일·업무목적·출발시간·출발지·도착시간·도착지·운행거리(Km)·차량유종. (L 유류대 / M 감가상각비 / S 합계는 양식 수식이 자동 계산)

## 권한 / 매니페스트 요점
- 위치(FINE/COARSE/BACKGROUND) · BLUETOOTH_CONNECT · POST_NOTIFICATIONS · FOREGROUND_SERVICE(+LOCATION + **CONNECTED_DEVICE**) · RECEIVE_BOOT_COMPLETED · REQUEST_IGNORE_BATTERY_OPTIMIZATIONS · REQUEST_INSTALL_PACKAGES · INTERNET
- ⚠️ **Android 14**: `connectedDevice` 타입 FGS엔 `FOREGROUND_SERVICE_CONNECTED_DEVICE` 권한 필수. 빠지면 startForeground가 SecurityException → 서비스가 조용히 종료되어 **자동·수동 감지 전부 먹통**이었음(v0.8에서 수정).
- 홈에서 권한 상태(초록/빨강 점) 표시 + 첫 실행 시 자동 요청.

## 알려진 이슈 / 주의
- 삼성 배터리 최적화 예외 안 하면 백그라운드 감지 놓침(홈에 배터리 상태/안내).
- 위치 "항상 허용" 권장.
- GPS 거리는 근사 정수 km(사업장 간은 고정거리).
- material-icons-extended로 dex가 큼(멀티덱스). 필요 아이콘만 써서 축소 가능.
- 디버그 빌드라 설치 시 "알 수 없는 출처" 경고 정상.

## 버전 히스토리(요약)
v0.1 뼈대 · v0.2 지도 핀(osmdroid) · v0.3 여비교통비 내보내기 · v0.4 사업장 시드+고정거리+작성 반복알림 · v0.5 안성2 주차장+반경150m · v0.6 이미연결 감지+수동시작+상태표시 · v0.7 **FGS 권한 버그 수정** · v0.8 **인앱 자동 업데이트** · v0.9 권한 상태 UI+브랜드 테마+경로/현위치 지도

## 다음 작업자(또는 다음 Claude)에게
- **소스: `D:\VehicleLog`** (이 저장소엔 README/APK만). 개발 PC 로컬 메모리에 빌드/배포 상세 기록 있음.
- 빌드/배포는 위 명령 사용. 버전 올리고 `release.ps1`.
- **TODO 후보**: 운행 합치기/분할 · 통계 화면 · 거리 소수점 옵션 · 사업장 핀 정밀화 UX · POI로 회사 양식 직접 채우기(현재는 붙여넣기) · 백그라운드 위치 안내 강화 · 아이콘 dex 축소 · (원하면) Play 내부테스트로 무탭 자동업데이트.
- **실차 테스트 필수**: BT 감지·GPS·자동분류는 실제 차량/주행에서만 검증 가능.
