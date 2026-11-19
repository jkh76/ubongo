# CLAUDE.md

우봉고(Ubongo) 솔로 타임어택 웹 게임. `index.html` 한 파일(HTML·CSS·JS)과 아이콘 파일(`favicon.ico`, `icons/`)로 동작하고, GitHub(jkh76/ubongo) 주소로 플레이한다.

## HTML 파일 이름 규칙

- 수정본이나 새 버전을 만들어 경훈에게 보여 줄 때는 파일 이름을 `우봉고.html`로 한다.
- 깃허브에 푸시할 때만 같은 내용을 `index.html`로 올린다. 저장소에는 `우봉고.html`을 따로 올리지 않는다.

## 작업 규칙

- 깃허브 푸시는 경훈이 "푸시해"라고 말할 때만 한다. 그 전에 수정한 `우봉고.html`을 먼저 보여 준다.
- 푸시할 때마다 `index.html`의 `const VERSION = 'v1.1 2026-11-20';`을 바꾼다.
  - 버전은 0.1씩 올린다 (예: 1.1 → 1.2). 경훈이 버전을 지정하면 그 값을 쓴다.
  - 날짜는 푸시하는 날짜(`yyyy-mm-dd`, 한국 시간). 홈·결과 화면 아래 `.version`에 표시된다.
  - 아래 '버전 기록' 표와 README.md의 버전 표에 한 줄씩 추가한다.
- 바닐라 JS + GSAP(+Draggable) + CSS. 외부 파일은 CDN의 GSAP·canvas-confetti·Google Fonts만 쓴다. 효과음은 Web Audio로 합성한다(음원 파일 없음).
- UI 문구는 한국어. 조각 모양·색과 디자인은 원작과 같게 한다. 보석은 아직 넣지 않았고, 솔로라 시간만 기록한다.
- 선택할 게 생기면 선택지를 보여 주고 고르게 한다.

## 게임 구성

- 모드: `time` 제한시간(3·5·10분, 많이 풀기) / `count` 스피드런(5·10·20문제, 빨리 풀기). 난이도: `easy` 조각 3개 / `hard` 조각 4개 / `mix` 3~4개.
- 패스: 10초 페널티(제한시간은 남은 시간에서 빼고, 스피드런은 기록에 더함).
- 조작: 드래그로 이동, 탭/R키 90° 회전, 조각을 든 채 좌우로 빠르게 흔들기 = 좌우 뒤집기, 상하로 흔들기 = 상하 뒤집기, 뒤집기 버튼/우클릭/F키. 원 그리기 회전은 쓰지 않는다.
- 우상단 ❚❚ 버튼: 일시정지 창(계속하기 / 그만하기). 그만하기는 두 번 눌러야 하고 그 판 기록은 저장하지 않는다. Esc·Enter·스페이스는 계속하기.

## 코드 구조 (`index.html`의 `<script>`, 번호 주석 순서)

1. `PIECE_DEFS`: 원작 조각 12종(A~L). `rotCW`·`flipH`·`flipV` 등 변환, `orientations`.
2. `generatePuzzle(k)`: 조각을 실제로 이어 붙여 판을 만들기 때문에 항상 답이 있다. 구멍 없는 모양만.
3. `SFX`: Web Audio 효과음.
4. `RecordStore` + `Sync`: 기록 저장(아래).
5. 게임 상태 `cfg`·`G`, 홈 설정(`bindSeg`), 전체 기록 창(`renderRecGrid`, `renderSyncBox`), 조각 SVG 렌더링, 배치(`canPlace`·`occupy`), 흔들기 인식(`SHAKE`, `shakeCheck`), Draggable, 타이머·결과(`endGame`, `showResult`), 일시정지(`quitGame`·`quitYes`·`quitNo`).

## 기록 저장

- 이 기기: `localStorage`의 `ubongo.best.<mode>.<setting>.<diff>` = `{solved, passes, timeMs, date}` (18칸), 최근 50판은 `ubongo.history`.
- 더 좋은 기록(`RecordStore.better`): 제한시간은 푼 문제가 많은 쪽(같으면 패스 적은 쪽), 스피드런은 시간이 짧은 쪽.
- 기기 간 공유 (v1.0): Firebase Realtime Database를 **SDK 없이 REST API**로 호출 (슬리더링크와 같은 방식).
  - 프로젝트 `ubongomobile`, `FB.key`(웹 API 키), `FB.db` = `https://ubongomobile-default-rtdb.asia-southeast1.firebasedatabase.app` (둘 다 공개돼도 되는 값).
  - 익명 로그인(Identity Toolkit `accounts:signUp`) 토큰을 `ubongo.fb.auth`에 두고, 만료되면 refresh token으로 갱신.
  - 데이터: `sync/<8자리 코드>` = `{best:{"<mode>_<setting>_<diff>":{s,p,t,d}}, updated}`. 코드 문자는 I·O·0·1을 뺀 32자. 이 기기 코드는 `ubongo.sync.code`.
  - `Sync.merge`: 서버를 받아 칸마다 더 좋은 기록을 양쪽에 남김(서버가 좋으면 이 기기에 쓰고, 이 기기가 좋으면 PATCH). 같은 걸 여러 번 해도 결과가 같아서 대기열이 필요 없다.
  - 앱 시작 0.6초 뒤, 기록 창 열 때, 신기록이 나왔을 때 `Sync.now()`. 오프라인이면 다음에 맞춤.
  - 보안 규칙(Firebase 콘솔): `sync/$code`만 로그인한 사용자가 읽기·쓰기, 코드·키·값 형식 검사, 다른 필드 금지.
  - claude.ai 미리보기에서는 외부 연결이 막힐 수 있음 → 실제 확인은 깃허브 주소에서.

## 테스트

- 문법: 마지막 `<script>`를 떼어 `node --check`.
- 화면·동기화: Playwright로 열고, GSAP/confetti CDN과 Firebase 주소를 가짜 응답으로 가로채(route) 두 기기(컨텍스트 2개)에서 코드 만들기 → 코드 입력 → 기록이 합쳐지는지 확인.

## 버전 기록

| 버전 | 날짜 | 내용 |
| --- | --- | --- |
| 1.1 | 2026-11-20 | 우상단 ✕를 두꺼운 일시정지 아이콘으로, 일시정지 창(계속하기 / 그만하기, 그만하기는 두 번 확인) |
| 1.0 | 2026-11-20 | 전체 기록 화면(18칸), 8자리 코드로 기기 간 기록 공유 (Firebase) |
| 0.5 | 2026-11-19 | 파비콘·홈 화면 아이콘 |
