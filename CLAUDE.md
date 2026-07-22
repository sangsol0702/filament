# FILAMENT 프로젝트 가이드

## 응답 스타일
- 모든 응답은 한국어 반말로 해줘.

## 프로젝트 개요
- 스타일 기반 브랜드/편집샵 큐레이션 웹사이트
- **단일 index.html SPA** — 모든 페이지·CSS·JS가 한 파일에 들어 있음
- 배포: GitHub 저장소 → Netlify 자동 배포 (fiament.netlify.app). main에 푸시하면 1~2분 내 반영됨

## 기술 스택
- 데이터: Google Sheets CSV export를 DB처럼 사용 (index.html 상단 `SHEET_ID` 상수, 시트명: `가게목록`, `스타일탐색`)
- Firebase (projectId: filament-752a8)
  - Firestore `events` 컬렉션 — 이벤트 신청/검수 (status: pending / approved / rejected)
  - Firestore `users` 컬렉션 — 구글 로그인 유저, role: user / admin / super_admin
  - Auth: Google 로그인 (signInWithPopup)
- 카카오맵 SDK — 지도 페이지, geocoder로 주소(`링크2` 컬럼)→좌표 변환

## index.html 내부 구조
- 페이지 div: `#discover` `#explore` `#community` `#events` `#map` `#admin` `#mypage` — 전부 body 직속, `.page` 클래스, `switchPage(id)`로 전환 (active 클래스 토글)
- 스크립트 5개 순서: 카카오 SDK 로더 → 메인(시트 로드·검색·정렬·switchPage·esc/safeUrl) → Firebase 이벤트 모듈(type=module) → 지도 → Firebase Auth 모듈(type=module)
- **중요**: module 스크립트 안의 함수는 반드시 `window.함수명 = 함수명`으로 노출해야 onclick 속성이나 switchPage의 typeof 체크에서 잡힘. 과거에 이거 빼먹어서 관리자탭이 "불러오는 중"에 멈추는 버그 있었음

## 컨벤션·주의사항
- 사용자 입력(이벤트 신청 데이터 등)을 innerHTML에 넣을 땐 반드시 `esc()` 로 이스케이프, URL은 `safeUrl()` (http/https만 허용) — XSS 방지
- onclick 속성에 문자열 데이터를 직접 보간하지 말 것 (따옴표 깨짐) — 인덱스나 id 참조 방식 사용 (focusShopByIdx 참고)
- Firestore 쿼리는 **where 1개만** 사용 — 복합 인덱스를 안 만들기 위해 종료일 필터·정렬은 클라이언트에서 처리
- Firestore 보안 규칙은 Firebase 콘솔에 배포돼 있음 (저장소의 firestore.rules 참고): events create는 status='pending' 강제, update/delete는 admin 이상, users role 변경은 super_admin만
- 첫 super_admin은 Firebase 콘솔에서 users 문서 role 필드를 직접 수정해서 만든 것
- 반응형: `@media (max-width: 768px)` 기준, 모바일 하단 탭바 `.mobile-tab` 사용
- PWA: manifest.json + 아이콘 4종(icon-192/512, maskable, apple-touch)이 저장소 루트에 있음. 파일명 바꾸면 index.html·manifest 참조가 깨짐
- 개인 코드 주석 태그: `// ejj EJJ`

## 알려진 미해결 이슈
- 카카오톡/인스타그램 인앱 브라우저에서 구글 로그인 차단됨 (구글의 웹뷰 정책) — 인앱 감지 시 외부 브라우저 유도 처리 붙이는 게 후보 작업
- 아이폰 홈화면 PWA(standalone)에서 signInWithPopup이 간헐적으로 문제될 수 있음
