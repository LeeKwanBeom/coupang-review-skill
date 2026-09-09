# 점검 기준선

점검일: 2026-09-09 (1회차 — 첫 기준선 개설, 진단만)
결함 3건 / 개선안 5건 / 인용불가로 제외 2건
직전 기준선 대비: 기준선 파일 없음 (해결 0 / 미해결 0 / 근거없음 0 / 신규 3)
점검 대상: SKILL.md 485행 (저장소 커밋 2437bbf) / 설치본·저장소 md5 동일
          (c2de499b2d2d0db647f552690b56ba79) / 실제 실행: 3매장 전부, Step 0~2 +
          결과 읽기까지 (Step 3 저장 미실행. 단 저장 스크립트는 엑셀 **사본**에
          dry-run) / 엑셀 백업: `$HOME/skillwork/쿠팡_저점수리뷰_backup_20260909_060811.xlsx`
          (세션 종료 시 소멸 — 개정안 참조) / 브라우저: Claude in Chrome, 시작 시
          로그인 상태(STATUS:200), 수집 중 401 없음
점검 모델·환경: Claude Fable 5.1, Cowork 클라우드 세션. 설치본 경로는
          `/root/.claude/skills/synced/<id>/coupang-review/SKILL.md`
          (점검표가 적은 `/mnt/skills/plugins/...` 는 이 환경에 없음)

행 번호는 모두 **485행짜리 저장소 SKILL.md(커밋 2437bbf) 기준**.

## 직전 기준선 판정

기준선 파일 없음. 이번 회차가 첫 기준선이다.

설치본·저장소 대조: md5 동일 → 재업로드 누락도 push 누락도 없음.

## 결함

| # | 심각도 | 위치(절·함수) | 행 | 문제 원문(그대로) | 실측/추론 | 왜 틀렸는지 | 수정 방향 |
|---|---|---|---|---|---|---|---|
| 1 | 중 | Step 3 동기화 정책 · 3-2 `too_old` | 303, 374 | `1. 날짜가 오늘 기준 1개월보다 이전 (수집 범위 밖)` / `too_old = d is not None and d < cutoff` | [실측] | API 의 `startDateTime` 은 **리뷰 작성일(`createdAt`)** 기준으로 거르는데, 엑셀 `날짜` 열은 **주문일(`orderedAt`)** 이다(Step 2 200행 `pick(r, 'orderedAt', ...)`). 오늘 수집분 580건 중 58건(10%)이 `orderedAt < startDate` 였다. 주문일이 컷오프보다 오래됐지만 리뷰는 범위 안인 저점수 리뷰는 매 회차 `too_old` 로 삭제된 뒤 `new` 로 다시 추가된다 → 변화가 없어도 `신규 1건, 삭제 1건` 으로 보고되고 최종 보고에 "신규 저점수 리뷰"로 매번 나열된다. 엑셀 사본 dry-run 으로 재현: 실제 변화 0건인데 `{"new": 1, "deleted": 1}` (김치찜의 정석 114SR5, 주문일 08-06 < 컷오프 08-09, 리뷰는 범위 안). 데이터 유실은 없다 — 오보고 문제 | 규칙 2(`missing`)만으로 범위 밖 행은 이미 지워지므로 규칙 1(`too_old`)은 `name in current` 인 매장에선 불필요하다. `too_old` 를 빼거나 `too_old and missing` 으로 좁힌다. 문서 303행 "(수집 범위 밖)" 도 같이 고친다. 열을 리뷰작성일로 바꾸는 건 사용자 결정 사항 |
| 2 | 하 | 결과 읽기 (인용 블록) | 261 | `매장 하나씩은 저점수가 몇 건이든 1,000자 안에 들어온다.` | [실측] | 오늘 매장별 JSON 길이 325 / 305 / 609자, 저점수 1건당 97~162자, 매장 메타데이터 약 200자. 저점수 6~7건이면 1,000자를 넘는다. 바로 다음 문단(300행)이 "그 매장마저 잘리면 … 3건씩" 이라고 반대 전제를 두고 있어 같은 절 안에서 서로 어긋난다. 또 같은 행의 `잘림은 오류 없이 조용히 일어나므로` 와 23행 `오류 없이 조용히 끊기므로`: 현재는 약 1,000자에서 끊기며 **끝에 `[TRUNCATED]` 표식이 붙는다** (ASCII·한글 모두 실측). 조용하지 않다 | "몇 건이든" 문장을 삭제하고 "저점수 5건 안팎이면 1,000자 근처, 넘으면 3건씩" 으로 고친다. 23행·261행·472행의 "조용히/오류 없이" 를 "`[TRUNCATED]` 표식으로 끝난다" 로 갱신 |
| 3 | 하 | 트러블슈팅 표 | 481, 482 | `\| \`응답 구조 불명\` \| API 응답 스키마 변경 \| 로그의 응답 키 목록으로 \`pick()\` 후보 추가 \|` / `\| \`apiTotal=0\`인데 화면엔 리뷰가 보임 \| \`statusType=EXPOSE\` 값 변경 가능성 \|` | [실측] | 이 API 는 오류를 **HTTP 200 + `code`≠`SUCCESS` + `error.message`** 로 돌려준다. 실측: 존재하지 않는 storeId → `{data:null, code:"10001", error:{message:"상점 정보를 찾을 수 없습니다."}}`, `statusType` 누락 → `{data:{statusType:...}, code:"10007", error:{message:"입력 된 정보가 유효하지 않습니다."}}`. `page()` 의 `j?.data ?? j?.result ?? j` 가 `data:null` 이면 봉투 전체를 돌려주고, 두 경우 다 148행 `응답 구조 불명: ["data","error","code"]` / `응답 구조 불명: ["statusType"]` 로 끝난다. 즉 (a) 481행의 원인·대응은 파라미터/매장 오류일 때 틀리고 — `pick()` 후보를 늘려도 안 낫는다 — (b) 482행이 말하는 `statusType` 문제는 `apiTotal=0` 이 아니라 `응답 구조 불명` 으로 나타난다 | 표에 "HTTP 200 이지만 `code`≠SUCCESS (`error.message` 확인)" 행을 추가하고 481·482행 문구를 실측대로 고친다. 코드 쪽은 개선안 1 |

## 개선안 (최대 5)

| # | 내용 | 이유 | 우선순위 |
|---|---|---|---|
| 1 | `page()` 에서 `j.code !== 'SUCCESS'` 면 `error.message` 를 담아 실패 처리 (`fail('API 오류 10001: 상점 정보를 찾을 수 없습니다')`) | 결함 3 의 코드 쪽. 지금은 API 가 이유를 말해주는데 버리고 "응답 구조 불명" 으로 보고한다 | 1 |
| 2 | 실행 전 백업을 Step 0 에 넣되 **사용자 폴더 안**(`$HOME/mnt/claude/backup/` 등)에 둔다 | 스냅샷 방식이라 실행이 행을 지우는데 절차에 백업이 없다. `$HOME/skillwork` 는 세션별 홈(`/sessions/rcw-<세션id>/`)이라 세션이 끝나면 못 꺼낸다(실측: 다른 세션의 skillwork 는 보이지 않음). 421행의 저장 실패 대체 경로(`~/skillwork/…backup.xlsx`)도 같은 이유로 세션이 끝나면 유실된다 | 2 |
| 3 | `apiTotal === null` 경로에서 `MAX_PAGES` 도달·페이지 실패를 `reason` 에 구분 표기 | 지금은 셋 다 128행 `총건수 확인 필요` 한 문장으로 합쳐진다(161행 루프가 200 에서 끝나도 표식 없음, 실패 시 `break` 해도 `failedPages` 검사보다 `apiTotal===null` 검사가 먼저라 가려짐). `ok=false` 라 저장은 막히니 결함은 아님 | 3 |
| 4 | 저장 후 엑셀을 다시 열어 행수·주문번호 집합이 `kept+new` 와 같은지 확인하는 한 줄 | 점검표 D. 이번 dry-run 처럼 사본 대상 검증이 쉬워서 비용이 작다 | 4 |
| 5 | `ratingFail` 이 전건이 아니어도 비율(예: 10%↑)이면 `ok=false`, 그리고 `collected !== apiTotal` 이면 ok 는 유지하되 보고에 경고 | 점검표 C·D. 오늘은 둘 다 0 / 정확히 일치라 실효는 없었다 | 5 |

## 실행 실측 기록

- 실행 시각: 2026-09-09 06:10:09 ~ 06:11:04 UTC (15:10 KST). 범위 `2026-08-09 ~ 2026-09-10`
- 실행 스크립트: Step 2 원문에 **audit-only 계측 1줄 추가** (`window._all[store.name] = all` 과 타이밍 기록. 판정 로직 변경 없음). 폴링 3회로 완료(첫 8초, 이후 20·18초)
- 매장별 (apiTotal / collected / ratingFail / 페이지 수 / 누적 소요 / 저점수 / 별점분포 1~5):
  - 김치찜의 정석 780573: 172 / 172 / 0 / 35p / 16s / 1건 / [0,0,1,1,170]
  - 참 제육 782948: 274 / 274 / 0 / 55p / 41s / 1건 / [1,0,0,3,270]
  - 퍽퍽살이 싫어 내가 만든 곱도리 987605: 134 / 134 / 0 / 27p / 54s / 3건 / [1,0,2,3,128]
  - 3매장 모두 `ok:true`, `failedPages:[]`, `authExpired:false`. orderReviewId·abbrOrderId 중복 0
- 저점수 5건 = 기존 엑셀 5행과 주문번호 완전 일치 (114SR5, 2VXGS1, 1FL4MU, 14D7ZC, 1R7HFZ). 신규 없음
- 응답 스키마 (page 1):
  - 최상위: `data`, `error`, `code` (정상: `error:null`, `code:"SUCCESS"`)
  - `data`: `content`, `pageNumber`, `pageSize`, `total`
  - 리뷰 18키: `orderReviewId, storeId, orderId, abbrOrderId, comment, memberId, images, replies, rating, statusType, tags, createdAt, modifiedAt, orderedAt, orderInfo, orderCount, customerName, orderType`
  - `rating` number(1~5) · `createdAt`/`orderedAt` `"2026-09-09T13:10:28.47"` 형식(로컬, TZ 없음) · `orderId` 19자리 숫자 · `abbrOrderId` 6자 · `orderInfo` 배열, 원소 키 `dishId, createdAt, dishName` · `statusType` 전건 `EXPOSE` · `orderType` `REGULAR`
  - `pick()` 후보 중 실제 존재: `total` · `content` · `rating` · `comment` · `orderedAt`(및 `createdAt`) · `abbrOrderId`(및 `orderId`) · `orderInfo`/`dishName`. 나머지 후보는 미존재(예비)
  - 후보에 없는 필드: `images, replies, tags, memberId, customerName, orderCount, orderType, modifiedAt, orderReviewId`
- 날짜 필터 기준 [실측]: `createdAt` 최소값이 3매장 모두 정확히 `2026-08-09`(=startDate), `orderedAt` 최소값은 07-15 ~ 07-21. → API 는 **리뷰 작성일** 로 거른다. `orderedAt < startDate` 건수: 15 / 25 / 18 (합 58)
- 마지막 페이지 다음(p28, p200): HTTP 200, `content:[]`, **`total:0`** — 총건수는 page 1 에서만 믿을 수 있다 (코드는 이미 그렇게 함)
- `size=10` 1회: HTTP 403 (본문 없음). 직후 `size=5` 는 200 정상 → WAF 세션 차단 없음
- 존재하지 않는 storeId(1): HTTP 200, `code:"10001"`. `statusType` 누락: HTTP 200, `code:"10007"`
- `javascript_tool` 반환 한도: ASCII·한글 모두 약 1,000자에서 끊기고 끝에 `[TRUNCATED]` 가 붙는다. 전체 `_res` JSON 1,243자(저점수 5건). 매장별 325 / 305 / 609자. 판정 요약 한 줄 104자
- 저장 스크립트 dry-run(엑셀 사본 `dryrun.xlsx`, 실제 파일 mtime 03:15 그대로): `{"new":1,"deleted":1,"rows":5}` — 결함 1 재현. 결과 5행은 기존과 동일

## 미확정으로 남긴 것

- `[TRUNCATED]` 표식이 2026-09-05 에도 붙었는지(도구 변경인지, 당시 문구가 "예외 없음" 뜻이었는지) 알 수 없음
- Step 1 의 `isLogin` 정규식이 실제 쿠팡이츠 로그인 URL 과 맞는지 — 로그아웃 없이는 확인 불가
- Step 1 주석 "결과에 URL 을 담으면 `[BLOCKED]`" — 재현하지 않음(막힐 위험이 있어 건드리지 않음)
- `statusType` 에 EXPOSE 외 어떤 값이 유효한지(숨김·삭제 리뷰가 어떤 값인지) 미확인

## 점검표 개정안

1. **설치본 경로 오류.** 점검표 39·51행과 README 의 `/mnt/skills/plugins/coupang-review/SKILL.md` 는 이 환경에 없다. 실제는 `/root/.claude/skills/synced/<id>/coupang-review/SKILL.md` (스킬 호출 시 "Base directory" 로 표시됨). md5 명령을 "스킬 호출 시 표시되는 Base directory 아래 SKILL.md" 로 바꿀 것
2. **백업 위치.** `$HOME/skillwork` 는 세션별 홈이라 세션이 끝나면 접근 불가(실측). 점검표 68~70행의 백업 지시를 사용자 폴더 안(`$HOME/mnt/claude/backup/`)으로 바꿀 것. 이번 회차 백업은 세션과 함께 사라진다 — 단 Step 3 을 안 돌려 실제 파일은 변경 없음
3. **상충 지시.** B 항목 "`size=5` 외의 값이 정말 403 인지 한 번만 실측" 과 판정 기준 "실제 엑셀과 WAF 는 건드리지 마라" 가 충돌. 이번엔 1회 실측(403 확인, 직후 정상)으로 처리. 한쪽으로 정리할 것 — 매 회차 403 을 확인할 필요는 없어 보임(기준선에 기록됐으니 "변경 의심 시에만" 으로)
4. **"되돌리면 안 되는 것" 표의 잘림 행** "한 번에 읽으면 약 1,000자에서 조용히 잘려" → "`[TRUNCATED]` 표식과 함께 잘려" 로. 실측 수치 항목(A) 의 "1,214자 / 7건" 도 이번 "1,243자 / 5건" 으로
5. **매 회차 통과만 나는 항목.** A "window.__API__ Step 1→2 인계" — Step 2 에 `|| '/api/v1/…'` 폴백이 있어 끊겨도 동작한다. 제거 후보
6. **배민에서 옮겨온 것 중 해당 없음.** E "배민의 `blocked` 집계처럼 `collected + α === apiTotal`" — 쿠팡 API 엔 blocked 개념이 없고 오늘 `collected === apiTotal` 정확 일치. 대신 "API 필터 기준(`createdAt`) vs 엑셀 날짜열(`orderedAt`) 일치 여부" 를 대조 항목으로 넣을 것
7. **신규 대조 항목.** "API 가 HTTP 200 + `code`≠SUCCESS 로 오류를 돌려주는 경우 스킬이 어떻게 보고하는지" 와 "응답 최상위 키·리뷰 키 목록(이번 기록) 과의 차이"
8. **계측 허용 범위 명시.** 이번에 Step 2 원문에 audit-only 1줄(`window._all`)을 넣어 원본 배열을 분석했다. 허용할지, 허용하면 "판정 로직은 건드리지 않는다" 조건을 점검표에 적을 것
9. **사본 dry-run 허용 명시.** 저장 스크립트를 엑셀 사본에 돌리면 실제 파일을 건드리지 않고 Step 3 을 검증할 수 있다(이번 결함 1 재현에 쓰임). "Step 3 은 실행하지 마라" 를 "실제 파일 대상으로는 실행하지 마라, 사본 dry-run 은 한다" 로
10. **토큰 안내.** 점검표 31행 "토큰은 대화에 남으니 작업이 끝나면 폐기하라고 안내" — 이번 회차 지시문에 토큰이 그대로 들어왔다. 채팅 보고 끝에 폐기 안내를 넣는다(이번 회차 수행)

## 다음 점검에서 대조할 것

- 결함 1~3 의 해결 여부 (수정 회차에서 사용자가 고른 것만)
- 응답 스키마: 위 최상위 3키 / `data` 4키 / 리뷰 18키 목록과 차이
- `createdAt` 최소값 = startDate 인지(필터 기준 유지 여부)
- `size=10` 은 재실측하지 않는다(개정안 3 채택 시). 변경 의심 시에만
- `[TRUNCATED]` 표식이 계속 붙는지
- 폴링 소요: 580건·119페이지에 54초. 리뷰가 두 배로 늘어도 폴링 40회(약 6.7분) 안
- 인용불가로 제외한 2건(isLogin 정규식, `[BLOCKED]` 주석)은 재현 방법이 생기면 확인

---

# 이전 기록

(없음 — 1회차)
