# theme-dashboard를 안드로이드 사이드로드 앱으로 패키징

**날짜:** 2026-07-16
**상태:** 승인됨

## 배경

`theme-dashboard`(https://jeon0jini82-ops.github.io/theme-dashboard/)는 GitHub Pages에 배포된 단일 `index.html` 정적 페이지로, 실시간 국내 주식 테마/대장주 데이터를 약 3분 주기로 갱신해 보여준다. PWA 관련 설정(manifest, 서비스워커, 아이콘)이 전혀 없는 순수 정적 페이지다.

목표는 이 사이트를 폰에 설치해서 아이콘을 눌러 바로 실행할 수 있는 안드로이드 앱(APK)으로 만드는 것. 용도는 **개인용 사이드로드**이며, 플레이스토어 정식 배포는 범위 밖이다 — 따라서 도메인 소유 인증(`assetlinks.json`)이나 정식 서명 키 관리는 다루지 않는다.

## 접근 방식

사이트 코드/데이터 로직은 손대지 않는다. PWA 껍데기(manifest + 아이콘 + 최소 서비스워커)만 추가해 정식 PWA로 만든 뒤, PWABuilder.com(무료 웹 도구, 로컬 JDK/Android SDK 설치 불필요)에서 이를 안드로이드 TWA APK로 패키징한다. 앱은 실행 시 라이브 사이트를 그대로 불러오므로, 실시간 데이터 갱신 동작은 변경되지 않는다.

검토한 대안:
- **PWA만 만들고 "홈 화면에 추가"로 끝내기** — 도구가 필요 없지만 결과물이 APK 파일이 아니라 폰에 종속된 바로가기라 채택하지 않음.
- **Bubblewrap CLI로 로컬 빌드** — 완전 자동화되지만 JDK+Android SDK(수 GB) 설치가 필요해 개인 사이드로드 용도엔 과함. 채택하지 않음.

## 추가/변경할 파일

```
theme-dashboard/
├── index.html          (수정: manifest link, theme-color, SW 등록 스크립트 추가)
├── manifest.json        (신규)
├── sw.js                (신규)
└── icons/
    ├── icon-192.png      (신규)
    └── icon-512.png      (신규)
```

### manifest.json

```json
{
  "name": "테마 내비게이션",
  "short_name": "테마내비",
  "start_url": "./",
  "scope": "./",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0a0b0d",
  "lang": "ko",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

### 아이콘

로컬 Python + Pillow로 생성 (외부 이미지 도구 불필요). 사이트 다크 네이비 배경(`#0a0b0d`) 위에 브랜드 블루(`#0052ff`)로 "테" 레터마크. maskable 용도로 안전하게 중앙 80% 안에만 그려서, `any`와 `maskable`을 같은 이미지로 겸용한다.

### sw.js

`index.html`만 캐싱하는 최소 shell 캐시. 라이브 시세 데이터 자체는 캐싱 대상이 아님(실시간성이 핵심이라 오프라인 캐싱 의미 없음). 오프라인 상태에서 최초 로드 시엔 캐시된 shell을 보여주되, 데이터 fetch가 실패하면 페이지 자체 스크립트가 처리하는 기존 동작을 그대로 따른다 — 서비스워커 레벨에서 별도 "오프라인 안내 화면"을 만들지는 않는다 (스코프 최소화).

### index.html 변경

`<head>`에 3줄 추가:
```html
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#0a0b0d">
<script>if('serviceWorker' in navigator) navigator.serviceWorker.register('sw.js');</script>
```

## 배포 및 패키징 절차 (핸드오프)

1. 위 파일들 커밋 후 `main` 브랜치에 push → GitHub Pages 자동 재배포
2. Chrome DevTools → Application 탭에서 manifest/서비스워커 인식 확인, Lighthouse PWA 감사 실행
3. PWABuilder.com에 사이트 URL 입력 → Android 패키지 생성 → APK 다운로드
4. 폰에서 "출처를 알 수 없는 앱 설치" 허용 후 다운로드한 APK 설치

3~4단계는 외부 웹 UI/폰 조작이 필요해 자동화 범위 밖이며, 사용자가 직접 진행하고 필요 시 캡처와 함께 안내받는다.

## 범위 제외

- 플레이스토어 정식 배포, `assetlinks.json` 도메인 인증, 릴리즈 서명 키 관리
- 오프라인 데이터 캐싱/동기화
- 사이트 자체의 UI/데이터 로직 변경
