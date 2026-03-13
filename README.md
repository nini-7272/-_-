# AGENTS.md — 샤오홍슈 데이터 수집 프로젝트

## 프로젝트 목적
샤오홍슈(小红书, xiaohongshu.com)에서 특정 키워드 검색 결과, 게시글 상세, 댓글 데이터를 **사람이 브라우징하는 동안 패시브하게 수집**하는 Chrome Extension 개발 및 운영.

## 핵심 원칙

### 반드시 지킬 것
- **추가 네트워크 요청 금지** — Extension은 브라우저가 이미 보내는 요청의 응답만 캡처. 자체적으로 API를 호출하면 차단 위험.
- **페이지 컨텍스트 무변조** — `window.fetch`, `XMLHttpRequest` 등을 덮어쓰지 않음. 보안 스크립트가 탐지함.
- **DevTools Network API만 사용** — `chrome.devtools.network.onRequestFinished`로 캡처. 이 방식은 사이트가 탐지 불가.

### 기술 스택
- Chrome Extension Manifest V3
- `chrome.devtools.network` API (DevTools 패널 방식)
- `chrome.storage.local` (월별 파티셔닝, unlimitedStorage)
- SpreadsheetML XML (.xls) 내보내기 (외부 라이브러리 없음)
- Vanilla JS only (프레임워크 없음)

## 디렉토리 구조
```
xiaohonghsu/
├── AGENTS.md              # 이 파일
├── SESSION_HANDOFF.md     # 이전 세션 상세 기록 (반드시 먼저 읽을 것)
├── network_data/          # API 요청/응답 샘플 + 수집 데이터
└── xhs-collector/         # Chrome Extension 소스코드
    ├── manifest.json
    ├── background.js      # 데이터 처리/저장 (월별 스토리지)
    ├── devtools.html/js   # DevTools 패널 등록
    ├── panel.html/js/css  # 실시간 캡처 로그 UI + 네트워크 리스너
    ├── popup.html/js/css  # 팝업 (통계 + 내보내기)
    └── export.js          # JSON/Excel 내보내기 (panel, popup 공유)
```

## 작업 시작 전 필수 확인
1. `SESSION_HANDOFF.md`를 먼저 읽어서 전체 맥락 파악
2. `xhs-collector/` 내 모든 소스 파일 읽기
3. `network_data/` 내 JSON 파일들로 실제 API 구조 확인

## 샤오홍슈 API 요약

| API | 용도 |
|-----|------|
| `edith.xiaohongshu.com/api/sns/web/v1/search/notes` | 검색 (POST) |
| `edith.xiaohongshu.com/api/sns/web/v1/feed` | 게시글 상세 (POST) |
| `edith.xiaohongshu.com/api/sns/web/v2/comment/page` | 댓글 (GET) |
| `edith.xiaohongshu.com/api/sns/web/v2/comment/sub/page` | 대댓글 (GET) |

- 검색 결과 `items[]` 배열의 인덱스 = 검색 순위
- 각 아이템의 `id` = `note_id` → 상세 페이지(`feed`)의 `note_id`와 동일 → 순위 매핑 키
- `xsec_token`은 노트별 고유 토큰, 검색 결과에서 발급되어 상세 조회 시 필수

## 안티스크래핑 주의사항
- `x-s`, `x-s-common` 서명은 난독화 JS가 브라우저에서 동적 생성 → 서버 재현 불가
- `shield/webprofile`로 브라우저 핑거프린팅 수행 → Headless 브라우저 탐지
- `redcaptcha` CAPTCHA 시스템 → 비정상 행위 시 자동 발동
- **우리 Extension은 이 모든 것을 우회하는 게 아니라, 정상 브라우징 트래픽을 관찰만 함**

## 알려진 이슈 (해결 필요)
1. **네트워크 리스너가 `panel.js`에 위치** — 패널 탭 클릭 전에는 캡처 안 됨. `devtools.js`로 옮기면 DevTools 열자마자 작동. 옮길 경우 panel과의 실시간 로그 통신 방식 설계 필요.
2. **Raw Log 용량** — 전체 API 응답 저장으로 커질 수 있음. 필요 시 요약만 저장하도록 변경 검토.

## 코딩 규칙
- 한국어 UI (패널, 팝업 텍스트)
- 변수/함수명은 영어 camelCase
- 외부 라이브러리 사용 금지 (Chrome Extension CSP 제약)
- `chrome.storage.local` 키 네이밍: `xhs_{타입}_{YYYY-MM}` (예: `xhs_posts_2026-03`)
