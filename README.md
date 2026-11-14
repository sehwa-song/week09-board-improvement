# WEEK 09 실습 2 · 검색·정렬·페이지네이션 게시판 개선

웹서비스설계 원본 게시판을 검증하고 정렬 오류와 커서 검색 누락을 개선한 프로젝트다. Node.js + Express, 단일 HTML/JavaScript 화면, 실행 시 생성되는 **1,000개 인메모리 게시글**로 구성한다. 실제 데이터베이스는 사용하지 않는다.

제출자: **컴퓨터인공지능학부 · 202346113 · 송세화**

- [보고서](https://github.com/sehwa-song/week09-board-improvement/raw/refs/heads/main/docs/%EC%B5%9C%EC%A2%85%EC%8B%A4%EC%8A%B5%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf): 보고서 원본 내려받기
- [보고서 Markdown 원문](docs/최종실습보고서.md): 원본·개선 비교와 사용자 촬영 캡처 42장
- [원본 캡쳐 기록](docs/캡쳐기록.md): 사용자 직접 검증 기록 보존본
- [GitHub 저장소](https://github.com/sehwa-song/week09-board-improvement)

## 실행 방법

Node.js와 npm을 설치한 뒤 다음 명령을 실행한다. 이번 점검 환경은 Node.js 24.21.0, 설치된 Express 5.3.0이다. `Object.hasOwn()`을 지원하는 Node.js 환경이 필요하다.

```bash
npm install
node server.js
```

브라우저에서 `http://localhost:3000`에 접속한다. `package.json`과 `package-lock.json`을 유지하며 `node_modules/`는 Git에서 제외한다. `npm start` 스크립트는 정의되어 있지 않다.

## 구조와 데이터 흐름

```text
index.html (검색·정렬·페이지 상태, fetch, 목록 표시)
    ↕ HTTP / JSON
server.js (Express API, 검색·정렬·페이지 분할)
    ↕ 배열 filter / sort / slice
POSTS (실행 시 생성되는 1,000개 게시글, 실제 DB 없음)

실습코드/
├── index.html
├── server.js
├── package.json
├── package-lock.json
├── README.md
└── docs/
    ├── 캡쳐기록.md
    ├── 최종실습보고서.md
    ├── 최종실습보고서.pdf
    └── images/                 # 사용자 촬영 PNG 42장
```

SQL 패널은 `updateSqlInspector()`가 만드는 **설명용 문자열**이다. 화면에 남아 있는 “실행 SQL”, “보안 파라미터 바인딩” 문구와 달리 실제 DB 실행·바인딩은 수행하지 않는다. 커서 SQL 미리보기에는 검색 조건도 반영되지 않으므로 실제 요청 URL과 JSON 응답을 기준으로 검증한다.

## API 명세

두 API 모두 `{ items, meta }`로 응답한다. 게시글 필드는 `id`, `title`, `content`, `author`, `views`, `createdAt`이다. 검색은 앞뒤 공백 제거·소문자 변환 후 제목 또는 본문에 `includes()`를 적용한다.

### GET /api/posts · 오프셋

| 파라미터 | 기본값 | 처리 |
|---|---|---|
| `page` | 1 | 10진수 정수 파싱, 0·음수·숫자 아님은 1, 마지막 페이지 초과는 마지막으로 보정 |
| `limit` | 10 | 양의 정수, 최대 100 |
| `sort` | latest | 직접 정의한 정렬 키 3개만 허용, 나머지는 latest |
| `search` | 빈 문자열 | 단일 문자열로 제목 또는 본문 검색 |

| 정렬 키 | 실제 기준 |
|---|---|
| `latest` | createdAt DESC, 같은 날짜는 id DESC |
| `oldest` | createdAt ASC, 같은 날짜는 id ASC |
| `views` | views DESC, 같은 조회수는 id DESC |

`meta`: `page`, `limit`, `totalCount`, `totalPages`, `hasNext`, `hasPrev`, `executionTimeMs`. `offset`은 응답 필드가 아니며 계산으로 구한다. 검색 결과가 없으면 `items=[]`, `page=1`, `totalPages=1`, `hasNext=false`, `hasPrev=false`다.

```text
/api/posts?page=2&limit=10&sort=views&search=redis
```

### GET /api/posts/cursor · 커서

| 파라미터 | 기본값 | 처리 |
|---|---|---|
| `cursor` | 없음 | 양수 정수이면 id < cursor, 없거나 파싱 불가·0·음수이면 조건 생략 |
| `limit` | 10 | 정수 파싱 후 1~50 범위, 0·파싱 불가는 10 |
| `search` | 빈 문자열 | 문자열만 받아 제목 또는 본문 검색 |

**ID 내림차순 고정**이며 생성 배열 순서를 유지한다. 날짜순·조회수순을 지정하는 기능은 없다. `meta`: `cursor`, `nextCursor`, `limit`, `hasNext`, `executionTimeMs`. 전체 개수·전체 페이지 수는 제공하지 않는다.

`nextCursor`는 반환한 마지막 ID, 결과가 없으면 null이다. 마지막 결과가 있으면 `hasNext=false`여도 마지막 ID가 남으므로 종료 판단에는 `hasNext`를 사용한다. 비양수 커서는 조건에 쓰이지 않지만 `meta.cursor`에 파싱값이 남을 수 있다.

```text
/api/posts/cursor?limit=10&search=B-Tree
/api/posts/cursor?limit=10&search=B-Tree&cursor=892
```

첫 응답의 `nextCursor=892`, 다음 응답은 `772`다. 검색·커서 필터를 모두 적용한 뒤 `limit+1`개를 확인한다.

## 원본과 직접 개선한 부분

| 항목 | 원본 | 개선 후 | Git 근거 |
|---|---|---|---|
| 정렬 키 | 객체 속성 접근으로 상속 속성 선택 가능 | 문자열 검사와 Object.hasOwn()으로 직접 정의한 키만 허용 | 3144d29 |
| 날짜 정렬 | ID만 비교하여 표시 날짜와 불일치 | createdAt 비교 우선, 같은 날짜는 ID 비교 | 19b3efc |
| 커서 검색 | 검색어 전달·필터 누락 | search 전달 → 제목/본문 필터 → 커서 필터 → limit+1 | f5ff3c8 |
| 커서 정렬 UI | 적용되지 않는 정렬 메뉴 표시 | 메뉴 숨김·비활성화, ID 내림차순 안내, 오프셋 복귀 시 복원 | f5ff3c8 |

```javascript
// 원본
const sortComparator = SORT_WHITELIST[sortKey] || SORT_WHITELIST.latest;
// 개선 후
const sortComparator =
  typeof sortKey === 'string' && Object.hasOwn(SORT_WHITELIST, sortKey)
    ? SORT_WHITELIST[sortKey]
    : SORT_WHITELIST.latest;
```

```javascript
// 원본: latest는 b.id - a.id, oldest는 a.id - b.id
// 개선 후
latest: (a, b) => b.createdAt.localeCompare(a.createdAt) || b.id - a.id,
oldest: (a, b) => a.createdAt.localeCompare(b.createdAt) || a.id - b.id
```

`__proto__`는 원본에서 비교 함수 대신 객체가 선택되어 TypeError가 발생했다. `constructor`는 상속 함수가 선택되어도 정상처럼 보일 수 있어 캡처와 소스 검증을 함께 사용했다. JavaScript 정렬 함수 선택 개선이며 실제 SQL Injection 방어 실험은 아니다.

오프셋 검색·페이지 보정, 조회수 동점 정렬, 디바운스, AbortController, URL 복원은 원본 기능이다. 디바운스는 코드 변경 없이 입력 간격을 통제해 재검증했다. 커서 검색 초기화도 기존 state.cursor 초기화와 응답 처리의 누적 배열 초기화 로직을 활용했다.

## 테스트 방법과 결과

아래는 사용자 캡처·기록에 근거한다. 과거 ID 정렬과 현재 날짜 정렬을 구분한다.

| 테스트 | 결과 | 근거 |
|---|---|---|
| B-Tree / 상세 본문 / 없는 검색어 | 84건·9페이지 / 1,000건·100페이지 / 빈 목록 | 그림 2~4, 17 |
| 날짜 정렬 | 원본 불일치, 개선 후 날짜·동점 ID 일치 | 5↔30, 6↔31 |
| 조회수 동점 | 당시 5,009회 ID 893→235→219 | 7 |
| invalid / __proto__ / constructor | 개선 후 latest로 대체 | 8~10, 28~29 및 소스 |
| page=0,-1,abc / limit=1000 | 1페이지 보정 / 최대 100개 | 11~14 |
| 2페이지 / 마지막 100페이지 | offset 10 / 990, 마지막 hasNext=false | 15~16, 원본 ID 정렬 당시 |
| 검색·정렬 변경 | 1페이지 초기화 | 17~18 |
| 커서 기본·더 보기·마지막 | nextCursor 991→981, 마지막 1·hasNext=false | 19~22 |
| 커서 검색·누적 초기화 | 검색 적용, 조건 변경 후 새 목록, 20개 누적 | 23↔33, 32~35 |
| 커서 전체 제목 검색 | 84건, API 9회, 중복 0, ID 1000~4 | 38, 백엔드 조회 범위 |
| 디바운스 | 수동 5건은 조건 미통제, 6회 연속 이벤트는 1건 | 24, 37 |
| 요청 취소·최신 결과 | 3G 이전 요청 취소, 최종 redis 200, AbortError 처리 | 25 두 장 |
| URL 복원 | 오프셋 redis·views·2페이지 새로고침·뒤로 가기 복원 | 26~27 |

2026-11-14 문서 점검에서는 등록된 서버 처리 함수를 프로세스 안에서 직접 실행해 **21개 보조 검증**을 통과했다. HTTP 통신이나 브라우저 조작 검증은 아니다. 커서 본문 검색도 1,000건·100회·중복 0·마지막 hasNext=false를 확인했으나 해당 조건의 새 사용자 캡처는 없다.

재현 시 서버 실행 후 API 주소의 items·meta를 확인한다. 브라우저 Network에서 입력 간격을 통제한 요청 수와 3G 취소 상태를 확인한다. 전체 커서 조회는 nextCursor를 다음 요청에 넣고 hasNext=false까지 반복하며 전체 수·ID 집합·순서를 비교한다. 상세 입력·기대·실제·판정은 보고서에 수록했다.

## 네 가지 필수 개념

1. **적합한 화면**: 오프셋은 번호로 특정 페이지에 이동하는 게시판에, 커서는 마지막 ID를 이어 받는 피드·더 보기에 적합하다. 현재 커서 UI는 버튼 방식이다.
2. **디바운스와 취소**: 디바운스는 마지막 입력 후 300ms 동안 추가 입력이 없을 때 호출한다. AbortController는 새 loadData() 시작 시 진행 중인 이전 요청을 취소하며 AbortError를 별도 처리한다. 입력 즉시가 아니라 다음 로딩 시작 시 취소한다.
3. **limit+1**: 검색·커서 조건에 맞는 11번째 후보가 있으면 10개만 반환하고 hasNext=true, 없으면 false다. 전체 개수 반환은 불필요하지만 현재 filter()는 후보 배열을 순회하므로 DB 인덱스 성능으로 해석할 수 없다.
4. **계산식**: `offset=(safePage-1)×limit`, `totalPages=max(1, ceil(totalCount/limit))`, `safePage=min(유효 page, totalPages)`. 10개씩 2페이지는 offset 10, 100페이지는 990이다. 1,000/10은 100페이지, 84/10은 9페이지이며 결과 0건도 구현상 1페이지다.

## 한계와 추가 개선점

- 메모리 데이터이며 조회수는 실행마다 무작위로 생성되어 캡처 값과 달라질 수 있다.
- executionTimeMs·X-Response-Time은 서버 처리 구간 관찰값이다. 화면 배지는 fetch·JSON 처리까지의 클라이언트 시간으로 범위가 다르다. DB 인덱스 효과·O(log N)·커서 속도 우위의 증거가 아니다.
- 커서 모드·누적 위치는 주소창에 저장하지 않는다. URL 복원 검증 범위는 오프셋의 검색어·정렬·페이지다. limit도 URL에서 복원하지 않는다.
- SQL 패널의 실행·바인딩 문구와 커서 검색 미리보기는 개선 대상이다. 이번 문서 작업에서 기능 코드는 변경하지 않았다.
- 그림 1은 기록의 최초 1페이지 설명과 달리 30페이지다. 그림 28 실제 파일명은 `fig28_sort_proto_fixed..png`이며 보고서에서 정확한 경로를 사용했다. 원본 기록·사진은 보존했다.
- 그림 9의 로컬 사용자 경로가 포함된 사진은 Git 추적 대상이다. 원격 공개 설정은 확인하지 않았다. 보고서에는 제출자 요청에 따라 학부·학번·이름을 기재했다.

## AI 활용과 직접 검증

AI는 코드 분석·개선 방향 정리, 문서 작성·근거 대조·보조 검증에 활용했다. 제출자는 브라우저·DevTools에서 직접 검증하고 42개 원본 캡처와 기록을 남겼다. 코드 개선은 기존 Git 이력에 반영되어 있으며 이번 작업은 문서화와 보조 검증이다. AI 도움 없이 모든 코드를 작성했다고 주장하지 않으며 새 캡처 생성이나 기존 사진 변조는 수행하지 않았다.
