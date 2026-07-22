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
  - Firestore `events` 컬렉션 — 이벤트 신청/검수 (status: pending / approved / rejected / revision)
    - `revision`(수정 요청)은 최종 상태가 아님 — 관리자탭에서 계속 승인·반려할 수 있고, 사유는 `revisionNote`에 저장
  - Firestore `users` 컬렉션 — 구글 로그인 유저, role: user / admin / super_admin
  - Firestore `posts` 컬렉션 — 커뮤니티 글 (uid, nickname, category, text, tags[], photo, likedBy[], createdAt)
  - Auth: Google 로그인 (signInWithPopup)
- 카카오맵 SDK — 지도 페이지, geocoder로 주소(`링크2` 컬럼)→좌표 변환

## index.html 내부 구조
- 페이지 div: `#discover` `#explore` `#community` `#events` `#map` `#admin` `#mypage` — 전부 body 직속, `.page` 클래스, `switchPage(id)`로 전환 (active 클래스 토글)
- 스크립트 6개 순서: 카카오 SDK 로더 → 메인(시트 로드·검색·정렬·switchPage·esc/safeUrl) → Firebase 이벤트 모듈(type=module) → 지도 → Firebase Auth 모듈(type=module) → 커뮤니티 글 모듈(type=module)
- **중요**: module 스크립트 안의 함수는 반드시 `window.함수명 = 함수명`으로 노출해야 onclick 속성이나 switchPage의 typeof 체크에서 잡힘. 과거에 이거 빼먹어서 관리자탭이 "불러오는 중"에 멈추는 버그 있었음
- 모듈 간 상태 공유는 `window._currentUser` / `window._currentRole` / `window._currentNickname` 전역으로 함 (Auth 모듈이 쓰고 커뮤니티 모듈이 읽음)

## 이벤트 신청 흐름
- 신청은 **로그인 필수** (openEventModal/submitEvent 양쪽에서 막음). 문서에 `uid`와 `applicantEmail`을 박아둠
- 마이페이지 "내 신청 현황"에서 본인 신청 건의 상태와 수정 요청 사유를 확인 — `where('uid','==',uid)` 하나만 쓰고 정렬은 클라이언트에서
- **로그인 필수로 바꾸기 전에 신청된 옛날 이벤트는 `uid`가 없어서 "내 신청 현황"에 안 뜸** (관리자탭에서는 정상적으로 보임)

## 관리자탭
- **진입점이 4개다**: 상단 네비 `.nl[data-p=admin]`의 li, 아바타 드롭다운 `#adminMenuLink`, 서랍 `.dn[data-p=admin]`, 모바일 하단탭 `.mnt[data-p=admin]`.
  전부 `applyAdminVisibility()`가 한꺼번에 토글함 — **하나라도 빠뜨리면 그리로 새어 들어감** (실제로 드롭다운만 막혀 있어서 나머지 3개로 비로그인 진입이 됐던 버그가 있었음)
- `switchPage('admin')` 자체에도 권한 가드가 있음. 콘솔에서 직접 호출해도 discover로 되돌림
- 권한 표시는 **켜기만 하지 말고 끄기도 해야 함** — 예전에 `display='block'`으로 켜기만 해서 관리자가 로그아웃한 뒤 일반 유저가 로그인하면 링크가 남아 있었음
- 이건 어디까지나 UI 차단이고 **진짜 방어선은 Firestore 규칙**. 클라이언트에서 `window._currentRole='admin'`으로 위조해도 껍데기만 열리고 데이터는 전부 막힘 (검증함)
- 규칙에 막혀 조회가 실패할 때 목록 컨테이너를 **반드시 채울 것** — 안 그러면 "불러오는 중..."에서 영원히 멈춤
- 상단 통계 4칸(검수 대기·승인 완료·반려·수정 요청)은 전부 `events`를 실제로 세서 채움 — 하드코딩 숫자였던 7/142/18/3은 제거됨
- 브랜드 입점·편집샵 등록 검수 카드는 목업이라 삭제함. 지금 검수 대상은 **이벤트 신청뿐**
- 관리자가 아니면 events 전체 조회가 규칙에 막힘 → 통계를 0으로 표시하지 말고 `—`로 둘 것 (0건과 "못 읽음"은 다름)
- 상태 변경 후에는 DOM만 갈아끼우지 말고 `loadAdminEvents()`를 다시 호출 — 안 그러면 상단 통계가 옛날 숫자로 남음

## 커뮤니티 (posts)
- 사이드바(카테고리·인기태그·활발한멤버)는 전부 목업이라 삭제함. 카테고리는 피드 상단 `.cat-chip` 필터로 대체 — 개수는 실제 글에서 셈
- 글 목록은 `orderBy('createdAt','desc') + limit(50)` 하나만 — 카테고리 필터는 클라이언트에서 처리(복합 인덱스 회피)
- 제목(`title`)은 **선택 입력** 60자. 없으면 카드에 제목 줄을 아예 안 그림. 제목·내용·사진이 전부 비면 게시 거부
- **검색도 클라이언트 전용** — Firestore는 부분 문자열 검색이 안 됨. 불러온 50건의 title·text·tags·nickname을 훑음.
  50건을 채우면 "최근 50건 안에서 찾았어요"를 같이 표시해서 전체 검색인 척하지 않음. 제대로 하려면 Algolia 같은 검색 서비스가 필요
- 검색어 강조 `hl()`은 **반드시 `esc()` 먼저 하고 그 결과에 `<mark>`만 끼워넣음**. 검색어에 `&<>"'` 가 있으면 엔티티(`&amp;`) 중간을 깨뜨릴 수 있어 강조를 생략함. 정규식 특수문자도 이스케이프해서 리터럴로 취급
- 검색과 카테고리는 AND. 카테고리 칩 개수도 검색 결과 기준으로 다시 셈 ("3건"인데 눌러보니 0건인 상황 방지)
- **사진 첨부는 Storage를 안 씀**. 캔버스로 긴 변 1000px·JPEG 0.72로 재압축해 dataURL을 Firestore 문서에 그대로 넣음 (문서 상한 1MiB라 900KB 넘으면 거부). 나중에 트래픽 늘면 Storage로 옮기는 게 맞음
- dataURL은 `safeUrl()`이 http/https만 허용해서 통과 못 함 → 커뮤니티 모듈 안의 `safeImg()`가 `data:image/(jpeg|png|webp);base64,...` 만 허용
- 닉네임은 글 문서에 복사해서 저장(읽을 때 users 조인 안 하려고). 그래서 `saveNickname()`이 `syncMyPostNicknames()`를 불러 기존 글 닉네임까지 갱신함
- 좋아요는 `likedBy` 배열에 uid를 arrayUnion/arrayRemove — 개수는 `likedBy.length`. 낙관적 갱신 후 실패하면 롤백. **전체 재렌더 대신 `updateLikeButton()`으로 버튼만 갈아끼움** (댓글 입력 중 포커스가 날아가서)
- 댓글은 `posts/{id}/comments` 서브컬렉션 — 글 문서에 배열로 넣으면 사진 dataURL이랑 합쳐서 1MiB를 넘길 수 있음. 💬 버튼 누를 때만 읽음
- 댓글 수는 글 문서의 `commentCount`에 `increment()`로 세어둠 (목록에서 서브컬렉션을 매번 읽지 않으려고)
- 댓글 삭제 권한: 댓글 작성자 본인 / 글 작성자 / 관리자
- 글 삭제 시 **댓글 서브컬렉션을 먼저 지움** — Firestore는 상위 문서를 지워도 서브컬렉션이 안 지워짐
- 작성 중인 댓글은 `_drafts[postId]`에 들고 있어서 재렌더에도 안 날아감
- 아직 없는 것: 무한스크롤, 태그 클릭 필터, 댓글 닉네임 동기화(닉네임 바꿔도 이미 쓴 **댓글**엔 예전 닉네임이 남음 — collectionGroup 인덱스가 필요해서 미룸)

## 컨벤션·주의사항
- 사용자 입력(이벤트 신청 데이터 등)을 innerHTML에 넣을 땐 반드시 `esc()` 로 이스케이프, URL은 `safeUrl()` (http/https만 허용) — XSS 방지
- onclick 속성에 문자열 데이터를 직접 보간하지 말 것 (따옴표 깨짐) — 인덱스나 id 참조 방식 사용 (focusShopByIdx 참고)
- Firestore 쿼리는 **where 1개만** 사용 — 복합 인덱스를 안 만들기 위해 종료일 필터·정렬은 클라이언트에서 처리
- Firestore 보안 규칙은 Firebase 콘솔에만 있고 저장소에는 파일로 없음 (콘솔 → Firestore → 규칙 탭이 유일한 원본). 요지: events create는 status='pending' 강제, update/delete는 admin 이상, users role 변경은 super_admin만, posts는 본인 글만 수정·삭제(관리자는 전부 삭제 가능)
- 첫 super_admin은 Firebase 콘솔에서 users 문서 role 필드를 직접 수정해서 만든 것
- 권한 부여는 두 군데: 마이페이지(최고관리자 전용, 닉네임 드롭다운으로 선택) + 관리자탭 사용자 목록(행별 드롭다운). 둘 다 이메일 입력 없이 닉네임으로 고름
- 반응형: `@media (max-width: 768px)` 기준, 모바일 하단 탭바 `.mobile-tab` 사용
- PWA: manifest.json + 아이콘 4종(icon-192/512, maskable, apple-touch)이 저장소 루트에 있음. 파일명 바꾸면 index.html·manifest 참조가 깨짐
- 개인 코드 주석 태그: `// ejj EJJ`
- **CSS 클래스 이름 충돌 주의** — 단일 파일 SPA라 모든 페이지가 같은 스타일시트를 공유함. 새 UI에 흔한 이름(`.search-bar`, `.card`, `.tag` 등)을 붙이기 전에 반드시 grep할 것.
  실제로 커뮤니티 검색바에 `.search-bar`를 썼다가 큐레이션 페이지의 `max-width:560px; margin:0 auto`가 딸려와 폭이 656→558px로 줄고 가운데 정렬된 적 있음.
  게다가 `.search-bar input`(0,1,1)이 `.search-input`(0,1,0)보다 우선순위가 높아 테두리까지 지워졌음 → 지금은 `.post-search-*` 접두사로 분리

## 알려진 미해결 이슈
- 카카오톡/인스타그램 인앱 브라우저에서 구글 로그인 차단됨 (구글의 웹뷰 정책) — 인앱 감지 시 외부 브라우저 유도 처리 붙이는 게 후보 작업
- 아이폰 홈화면 PWA(standalone)에서 signInWithPopup이 간헐적으로 문제될 수 있음
