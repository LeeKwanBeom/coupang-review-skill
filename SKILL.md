---
name: coupang-review
description: "쿠팡이츠 사장님 포털에서 저점수(1~3점) 리뷰를 자동 수집해 엑셀에 저장하는 스킬. 사용자가 \"쿠팡 리뷰\", \"쿠팡이츠 리뷰\", \"저점수 리뷰 확인\", \"coupang review\", \"쿠팡 리뷰 조회\", \"쿠팡 리뷰 해줘\" 등 쿠팡이츠 리뷰와 관련된 요청을 할 때 반드시 이 스킬을 사용할 것. 크롬의 인증 컨텍스트에서 내부 API를 호출해 3개 매장의 최근 1개월 리뷰를 수집하고, \"전체\" 단일 시트 엑셀에 저장한다."
---
# 쿠팡이츠 저점수 리뷰 수집

## 매장 정보

| 매장명 | storeId |
|---|---|
| 김치찜의 정석 | 780573 |
| 참 제육 | 782948 |
| 퍽퍽살이 싫어 내가 만든 곱도리 | 987605 |

URL: `https://store.coupangeats.com/merchant/management/reviews/<storeId>`

## 설계 근거 (건드리기 전에 읽을 것)

- **API를 쓰는 이유**: `/api/v1/merchant/reviews/search`는 포털과 **same-origin**이라 페이지 컨텍스트에서 그대로 호출된다. UI 조작·스크롤이 전혀 필요 없다. (배민은 API가 다른 오리진이라 브라우저 확장 스크립트에서 전부 차단된다 — 두 스킬의 구조가 다른 이유다.)
- **`size=5` 고정**: 다른 값을 넣으면 WAF가 403을 돌려준다. 페이지 수가 늘더라도 이 값을 바꾸지 말 것.
- **매장·페이지 모두 순차 호출**: 병렬로 부르면 API가 잘못된 `total`을 반환하는 사례가 있었다.
- **`javascript_tool` 호출은 45초에서 CDP 타임아웃**이 난다. 수집은 백그라운드로 돌리고 폴링으로 확인한다.
- **`javascript_tool` 반환값은 약 1,000자에서 잘린다**(2026-09-05 실측, 2026-09-09 재확인). 예외는 없고 끝에 `[TRUNCATED]` 표식과 함께 끊기므로 결과는 처음부터 매장 단위로 나눠 읽는다(아래 "결과 읽기").
- **엑셀은 사용자 PC에서 직접 처리**한다(`device_bash`). 클라우드 컨테이너엔 `/sessions` 경로 자체가 없으므로 `find /sessions/*/mnt/...` 같은 탐색을 되살리지 말 것. 스테이징·전송·커밋 왕복도 필요 없다.

### 조용한 0건을 막는 세 가지 원칙 (2026-09-06 도입 — 되돌리지 말 것)

이 스킬의 가장 위험한 실패는 "조용히 0건"이다. 리뷰가 없어서 0건인지, 필드명이 바뀌어서 0건인지 구분하지 못하면 **"저점수 리뷰 없음"이라는 잘못된 안심**을 주고, 이어지는 저장 단계가 기존 엑셀 행까지 지운다. 아래 셋은 그 사고를 막는 장치다.

1. **필드를 못 찾으면 0이 아니라 `null`이다.** `pick()` 결과에 `|| 0`을 붙이지 말 것. `null`은 "확인 필요"를 뜻하고, 그 매장은 저장 단계에서 **읽지도 쓰지도 않는다**(기존 행 보존).
2. **`rating`이 숫자가 아니면 조용히 건너뛰지 않는다.** `ratingFail`로 세고, 그 리뷰는 `[별점확인필요]`를 붙여 저점수 목록에 남기며, 최종 보고에 건수를 올린다. 4~5점으로 **확인된** 리뷰만 제외한다.
3. **매장 성공/실패는 `error` 필드 유무가 아니라 `ok` 플래그로 판정한다.** `ok: true`는 "총건수를 읽었고 · 모든 페이지를 받았고 · 수집률이 98% 이상이고 · 401이 없었다"를 전부 만족할 때만 붙는다. 기본값은 실패다. 저장 스크립트는 `ok !== true`인 매장을 통째로 건너뛴다.

---

## Step 0: 환경 확인

```bash
ls -d $HOME/mnt/claude && python3 -c "import openpyxl;print('openpyxl ok')" && ls $HOME/mnt/claude/'~$쿠팡_저점수리뷰.xlsx' 2>/dev/null && echo LOCKED || echo UNLOCKED
```

- `LOCKED` → 엑셀을 닫아달라고 요청하고 재확인한다.
- `device_bash` 자체가 실패하면 → PC에 연결되어 있지 않다고 알리고, 수집은 진행하되 결과를 채팅 표로만 출력한다.

엑셀 경로는 `$HOME/mnt/claude/쿠팡_저점수리뷰.xlsx` 이다.

`UNLOCKED`면 **실행 전 백업**을 사용자 폴더 안에 남긴다. 이 스킬은 스냅샷 동기화라 Step 3가 기존 행을 지우므로 백업이 유일한 되돌리기 수단이다. `$HOME/skillwork`나 `$HOME` 아래는 세션별 홈이라 세션이 끝나면 접근할 수 없으니(2026-09-09 실측) 거기에 두지 말 것.

```bash
mkdir -p $HOME/mnt/claude/backup && if [ -f "$HOME/mnt/claude/쿠팡_저점수리뷰.xlsx" ]; then cp "$HOME/mnt/claude/쿠팡_저점수리뷰.xlsx" "$HOME/mnt/claude/backup/쿠팡_저점수리뷰_$(date +%Y%m%d_%H%M%S).xlsx" && echo "백업 완료: $(ls -t $HOME/mnt/claude/backup/ | head -1)" || echo "백업 실패 — 중단하고 사용자에게 알린다"; else echo "백업 대상 없음(첫 실행)"; fi
```

백업은 `backup/` 폴더에 회차마다 쌓인다. 정리는 사용자 몫이며 스킬이 지우지 않는다.

---

## Step 1: 탭 열기 + 로그인 확인

```
tabs_context_mcp(createIfEmpty=true)
navigate(url=https://store.coupangeats.com/merchant/management/reviews/780573, tabId=<탭ID>)
```

페이지 로드 후 한 번에 확인한다.

```javascript
window.__API__ = '/api/v1/merchant/reviews/search';   // 엔드포인트 변경 시 이 줄만 고치면 Step 2도 함께 반영된다
// 결과에 URL(쿼리스트링)을 담지 않는다 — 담으면 브라우저 도구가 [BLOCKED]로 결과 자체를 막아
// 실제 상태 코드(401 등)를 못 본다. 상태 코드만 돌려준다.
const st = await fetch(window.__API__ + '?storeId=780573&page=1&size=5',
  { credentials: 'same-origin', headers: { Accept: 'application/json' } })
  .then(r => 'STATUS:' + r.status).catch(e => 'FETCH_ERROR:' + e.message);
JSON.stringify({ st, url: location.href, isLogin: /login|signin|auth|oauth|account|member/i.test(location.href),
  hasPw: !!document.querySelector('input[type=password]'), preview: document.body.innerText.substring(0, 150) })
```

**중단 조건** — `isLogin` / `hasPw` / `st === 'STATUS:401'`:
> 쿠팡이츠 스토어에 로그인이 필요합니다. 크롬에서 로그인 후 '완료했어요'라고 알려주세요.

사용자 확인 후 Step 1부터 재시작한다.

`STATUS:403`은 WAF 제약일 수 있으니 중단하지 않고 Step 2로 진행한다. 결과가 `[BLOCKED:`로 시작하면 무시하고 진행한다.

> 이 확인은 **시작 시점 한 번뿐**이다. 수집 도중 세션이 풀리는 경우는 Step 2가 401을 직접 잡아 그 매장을 실패로 돌린다(아래).

---

## Step 2: 3개 매장 순차 수집

날짜 범위는 오늘 기준 최근 1개월. 월말 overflow는 해당 월 마지막 날로 클램핑한다(8/31 − 1개월 = 7/31).

```javascript
window._res = null; window._err = null; window._log = [];
(async function () {
  const LOG = m => window._log.push(new Date().toISOString().substring(11, 19) + ' ' + m);
  const pick = (o, ...ks) => { for (const k of ks) if (o?.[k] != null) return o[k]; };   // 못 찾으면 undefined — 절대 || 0 을 붙이지 말 것
  const num = v => typeof v === 'number' ? v : (typeof v === 'string' && /^\d+(\.\d+)?$/.test(v.trim()) ? +v : null);
  const stores = [
    { id: '780573', name: '김치찜의 정석' },
    { id: '782948', name: '참 제육' },
    { id: '987605', name: '퍽퍽살이 싫어 내가 만든 곱도리' }
  ];
  const pad = n => String(n).padStart(2, '0');
  const fmt = d => `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
  const today = new Date();
  const monthsBack = (d, m) => {
    const i = d.getMonth() - m, y = d.getFullYear() + Math.floor(i / 12), mo = ((i % 12) + 12) % 12;
    return new Date(y, mo, Math.min(d.getDate(), new Date(y, mo + 1, 0).getDate()));
  };
  const startDate = fmt(monthsBack(today, 1));
  const endDate = fmt(new Date(today.getTime() + 864e5));
  LOG(`범위 ${startDate} ~ ${endDate}`);

  const API = window.__API__ || '/api/v1/merchant/reviews/search';
  const TIMEOUT = 15000;    // 서버 무응답 시 무기한 대기 방지
  const MAX_PAGES = 200;    // 총건수를 못 읽었을 때만 쓰는 안전 상한

  async function page(storeId, p) {
    const url = `${API}?storeId=${storeId}&page=${p}&statusType=EXPOSE&startDateTime=${startDate}&exclusiveEndDateTime=${endDate}&size=5`;
    for (let a = 1; a <= 2; a++) {
      const ac = new AbortController(); const t = setTimeout(() => ac.abort(), TIMEOUT);
      try {
        const r = await fetch(url, { credentials: 'same-origin', headers: { Accept: 'application/json' }, signal: ac.signal });
        clearTimeout(t);
        // 401은 재시도해도 소용없다. 즉시 치명 실패로 올린다. 403은 WAF 제약일 수 있어 일반 오류로 다룬다.
        if (r.status === 401) return { ok: false, error: 'HTTP 401', fatal: true };
        if (!r.ok) throw new Error('HTTP ' + r.status);
        const j = await r.json();
        // 이 API는 오류를 HTTP 200 + code≠SUCCESS + error.message 로 돌려준다(2026-09-09 실측: 없는 storeId → 10001,
        // statusType 누락 → 10007). 재시도해도 같은 답이므로 즉시 실패로 올린다. code 필드가 없는 응답은 종전대로 다룬다.
        if (j && typeof j === 'object' && 'code' in j && j.code !== 'SUCCESS')
          return { ok: false, apiError: true, error: `API 오류 ${j.code}: ${j?.error?.message || '(메시지 없음)'}` };
        return { ok: true, data: j?.data ?? j?.result ?? j };
      } catch (e) {
        clearTimeout(t);
        const msg = e.name === 'AbortError' ? `TIMEOUT(${TIMEOUT}ms)` : e.message;
        if (a === 2) return { ok: false, error: msg };
        await new Promise(s => setTimeout(s, 500));
      }
    }
  }

  const contentOf = d => pick(d, 'content', 'reviews', 'items', 'list');

  async function one(store) {
    LOG(`[${store.name}] 시작`);
    // 기본값은 실패다. ok는 아래 검사를 전부 통과해야만 true가 된다.
    const fail = (reason, extra) => {
      LOG(`[${store.name}] 확인필요 — ${reason}`);
      return Object.assign({ storeName: store.name, ok: false, reason, startDate, endDate,
        apiTotal: null, collected: 0, ratingFail: 0, failedPages: [], authExpired: false,
        dist: [0, 0, 0, 0, 0], lowScore: [] }, extra || {});
    };

    const first = await page(store.id, 1);
    if (!first.ok) return fail(first.fatal ? '인증만료(401) — 수집 도중 세션 종료'
      : first.apiError ? first.error : 'p1 실패 ' + first.error,   // API 오류는 서버가 준 메시지를 그대로 사유로 쓴다
      { authExpired: !!first.fatal });
    const d = first.data;
    const content = contentOf(d);
    if (!content) return fail('응답 구조 불명: ' + JSON.stringify(Object.keys(d || {})));

    // 총건수를 못 읽으면 0이 아니라 null이다. 0으로 접으면 pages=1이 되어 5건만 긁고 끝난다.
    const apiTotal = num(pick(d, 'total', 'totalCount', 'totalElements'));
    LOG(`[${store.name}] apiTotal=${apiTotal === null ? 'null(확인필요)' : apiTotal}`);

    const all = [...content];   // page 1 응답 재사용 (중복 fetch 제거)
    const failedPages = [];
    let authExpired = false;

    if (apiTotal === null) {
      // 총건수 불명 — 빈 페이지가 나올 때까지 계속 받는다. 결과는 어차피 '확인필요'로 나가지만
      // 수집 자체는 끝까지 해서 사용자가 눈으로 볼 저점수를 놓치지 않는다.
      for (let p = 2; p <= MAX_PAGES; p++) {
        const r = await page(store.id, p);
        if (!r.ok) { failedPages.push(p); if (r.fatal) authExpired = true; break; }
        const c = contentOf(r.data) || [];
        if (!c.length) break;
        all.push(...c);
        if (p % 10 === 0) LOG(`[${store.name}] ${p}/? (총건수 불명)`);
      }
    } else {
      const pages = Math.max(1, Math.ceil(apiTotal / 5));
      LOG(`[${store.name}] pages=${pages}`);
      for (let p = 2; p <= pages; p++) {
        const r = await page(store.id, p);
        if (r.ok) all.push(...(contentOf(r.data) || []));
        else { failedPages.push(p); if (r.fatal) { authExpired = true; break; } }
        if (p % 10 === 0) LOG(`[${store.name}] ${p}/${pages}`);
      }
    }
    LOG(`[${store.name}] 수집 ${all.length}건 (실패 페이지 ${failedPages.length})`);

    const dist = [0, 0, 0, 0, 0];
    let ratingFail = 0;
    const norm = v => {   // 숫자 epoch 등 비-ISO 형식도 YYYY-MM-DD로 정규화
      if (!v) return '';
      if (typeof v === 'number') { const dt = new Date(v < 1e12 ? v * 1000 : v); return isNaN(dt) ? '' : dt.toISOString().substring(0, 10); }
      const s = String(v);
      if (/^\d{4}-\d{2}-\d{2}/.test(s)) return s.substring(0, 10);
      const dt = new Date(s); return isNaN(dt) ? s.substring(0, 10) : dt.toISOString().substring(0, 10);
    };
    const low = [];
    for (const r of all) {
      const v = num(pick(r, 'rating', 'score', 'starCount'));
      const rated = v !== null && v >= 1 && v <= 5;          // 별점을 '확인'했는가
      if (rated) dist[Math.round(v) - 1]++; else ratingFail++;
      if (rated && v > 3) continue;   // 4~5점으로 확인된 것만 제외한다. 못 읽은 별점은 남긴다.
      const info = pick(r, 'orderInfo', 'items', 'menuInfo', 'orderItems') || [];
      const menu = Array.isArray(info)
        ? info.map(o => typeof o === 'string' ? o : (pick(o, 'dishName', 'menuName', 'name') || '')).filter(Boolean).join(', ')
        : (info && typeof info === 'object' ? (pick(info, 'dishName', 'menuName', 'name') || '') : String(info || ''));
      const date = norm(pick(r, 'orderedAt', 'orderDate', 'createdAt', 'reviewedAt')) || '날짜없음';
      const raw = pick(r, 'abbrOrderId', 'orderId', 'shortOrderId', 'orderNo');
      const orderNo = raw ? String(raw) : `NOKEY_${date}_${menu.substring(0, 15)}`;   // orderNo 필드가 사라진 경우 보조키
      if (!raw) LOG(`[${store.name}] orderNo 없음 → 보조키 ${orderNo}`);
      low.push({ stars: rated ? Math.round(v) : null, orderDate: date, orderNo, menu,
        reviewText: (rated ? '' : '[별점확인필요] ') +
          ((pick(r, 'comment', 'reviewContent', 'content', 'text') || '').trim() || '(텍스트 없음)') });
    }

    // ok는 아래를 전부 통과할 때만 true. 하나라도 걸리면 저장 단계가 이 매장을 건드리지 않는다.
    let ok = true, reason = '';
    if (authExpired)                                  { ok = false; reason = '인증만료(401) — 수집 도중 세션 종료'; }
    else if (apiTotal === null)                       { ok = false; reason = '총건수 확인 필요 — total 계열 필드를 못 읽음'; }
    else if (failedPages.length)                      { ok = false; reason = `페이지 실패 p${failedPages.join(',p')}`; }
    else if (apiTotal > 0 && all.length < apiTotal * 0.98) { ok = false; reason = `수집률 낮음 ${all.length}/${apiTotal}`; }
    else if (all.length > 0 && ratingFail === all.length)  { ok = false; reason = '별점 필드 확인 필요 — 전건 파싱 실패'; }
    if (ok) LOG(`[${store.name}] 정상 완료 ${all.length}건 (저점수 ${low.length})`);
    else LOG(`[${store.name}] 확인필요 — ${reason}`);

    return { storeName: store.name, ok, reason, startDate, endDate,
      apiTotal, collected: all.length, ratingFail, failedPages, authExpired, dist, lowScore: low };
  }

  try {
    const out = [];
    for (const s of stores) { out.push(await one(s)); await new Promise(r => setTimeout(r, 500)); }
    window._res = out; LOG('전체 완료 (ok ' + out.filter(s => s.ok).length + '/' + out.length + ')');
  } catch (e) { window._err = e.message; LOG('오류 ' + e.message); }
})();
'수집 시작'
```

### 완료 폴링

첫 확인은 5초 후, 이후 10초 간격.

```javascript
window._err ? 'ERROR:' + window._err
  : window._res ? 'DONE'
  : 'LOG:' + JSON.stringify(window._log.slice(-4))
```

- `ERROR:` → 사용자에게 알리고 중단.
- `LOG:` → 진행 중. 매장·페이지 진행 상황을 보며 계속 대기.
- 최대 40회. 그 이상이면 `window._log` 전체를 보고한다.
- **폴링 호출 자체가 도구 오류를 반환하는 경우**(예: 브라우저 미연결)는 페이지 안의 수집과 무관하다. 페이지의 JS는 계속 돌고 있으므로 데이터 손실 없이 같은 폴링을 그대로 재시도하면 되고, 이 오류는 40회 상한에 포함하지 않는다.

### 결과 읽기

**먼저 판정 요약 한 줄을 읽는다.** 어느 매장이 저장 대상인지 여기서 결정된다.

```javascript
JSON.stringify(window._res.map(s => [s.storeName, s.ok, s.reason, s.apiTotal, s.collected, s.ratingFail, s.lowScore.length]))
```

**그다음 매장 단위로 3번 나눴 읽는다.** 한 번에 전체를 읽지 않는다.

```javascript
JSON.stringify(window._res[0])   // 이어서 [1], [2]
```

> **왜 나눴 읽는가**: `javascript_tool` 반환값은 약 1,000자에서 잘린다. 2026-09-09 실측에서 저점수가 3개 매장 합계 5건뿐인 결과도 전체 JSON이 1,243자였다(2026-09-05에는 7건에 1,214자) — 매장별 메타데이터가 약 200자씩 붙어 금방 한계를 넘는다. 잘림은 예외 없이 끝에 `[TRUNCATED]` 표식과 함께 일어나므로, 표식이 보이면 그 결과는 버리고 더 잘게 읽는다. 매장 하나는 메타데이터 약 200자 + 저점수 1건당 약 100~160자라, 저점수 5건 안팎이면 1,000자 근처이고 그 이상이면 아래처럼 3건씩 읽는다.

매장 하나의 저점수가 유난히 많아 그 매장마저 잘리면(끝에 `[TRUNCATED]` 표식), 그때만 배열을 더 쪼개 읽는다:
```javascript
JSON.stringify({s: window._res[0].storeName, r: window._res[0].lowScore.slice(0, 3)})   // 3건씩
```

**판정**

- `ok === true` → 저장 대상. 정상 수집이 확인된 매장이다.
- `ok === false` → **확인 필요**. 사용자에게 `reason`을 그대로 알리고, 나머지 매장은 계속 진행한다. 이 매장은 저장 스크립트가 건너뛰므로 기존 엑셀 행이 지워지지 않는다.
- `authExpired === true`가 하나라도 있으면 → 수집 도중 세션이 끊긴 것이다. 최종 보고에 반드시 올리고 이렇게 안내한다:
  > ⚠️ 수집 도중 로그인이 풀렸습니다([매장명]). 크롬에서 다시 로그인한 뒤 '완료했어요'라고 알려주시면 해당 매장만 다시 수집합니다.
- `ratingFail > 0`인데 `ok === true` → 일부 리뷰만 별점을 못 읽은 것이다. 진행하되 건수를 최종 보고에 올린다. 그 리뷰는 `[별점확인필요]`가 붙어 엑셀에 남는다.
- **3개 매장 모두 `apiTotal === 0`** → 리뷰가 정말 없을 수도 있지만 파라미터가 안 먹었을 가능성이 더 크다. 이상 신호로 보고 사용자에게 "쿠팡이츠 화면에 최근 1개월 리뷰가 보이는지" 확인을 요청한 뒤 저장한다.
- 한 매장만 `apiTotal === 0`이고 `ok === true` → 정상으로 본다(그 매장은 실제로 리뷰가 없는 것).

---

## Step 3: 엑셀 저장

### 3-1. JSON 기록

특수문자가 들어갈 수 있으므로 반드시 따옴표로 감싼 heredoc을 쓴다. 위에서 나누어 읽은 매장 3개를 **`ok === false`인 매장까지 그대로 포함해** 하나의 배열로 합쳐 기록한다. 저장 스크립트가 `ok` 플래그를 보고 스스로 건너뛴다 — 사람이 골라내지 않는다.

각 매장 객체에 최소한 `storeName` · `ok` · `reason` · `apiTotal` · `collected` · `ratingFail` · `lowScore`가 들어가야 한다.

```bash
mkdir -p $HOME/skillwork && cat > $HOME/skillwork/coupang.json << 'ENDJSON'
[{"storeName":"...","ok":true,"reason":"","apiTotal":0,"collected":0,"ratingFail":0,"lowScore":[]}, ...]
ENDJSON
```

기록 직후 `python3 -c "import json;json.load(open(...))"`로 파싱되는지 한 번 확인한다 — 읽기 단계에서 잘린 조각을 붙였다면 여기서 잡힌다. 다만 **파싱 검증은 잘림만 잡는다.** 빈 `lowScore`가 진짜인지 실패인지는 `ok` 플래그만이 구분한다.

`$HOME/skillwork`는 사용자에게 보이지 않는 세션별 작업 공간이다(세션이 끝나면 사라진다 — 그래서 백업은 여기 두지 않는다). 사용자 폴더에 임시 파일을 만들지 않는다.

### 3-2. 저장 스크립트

시트: **"전체" 단일 시트**. 열: `매장명 | 날짜 | 별점 | 주문번호 | 주문메뉴 | 리뷰내용`. 날짜는 `YYYY-MM-DD`(배민 스킬과 동일 형식).

동기화 정책 — **`ok === true`인 매장에만 적용한다.** 그 매장의 행 중 **이번 수집 결과에 주문번호가 없는 행**을 삭제한다. 범위 밖으로 밀려난 리뷰, 삭제된 리뷰, 4~5점으로 수정된 리뷰가 전부 여기에 들어간다.

**엑셀의 `날짜` 열로는 삭제를 판정하지 않는다.** API의 `startDateTime`은 **리뷰 작성일(`createdAt`)** 기준으로 거르는데 `날짜` 열은 **주문일(`orderedAt`)** 이라 두 필드가 다르다(2026-09-09 실측: 수집 580건 중 58건이 주문일은 범위 밖, 리뷰는 범위 안). 예전처럼 `날짜 < 오늘−1개월` 행을 지우면 그런 리뷰가 매 회차 삭제→재추가되어 변화가 없어도 "신규 1건, 삭제 1건"으로 오보고된다. 주문번호 대조만으로 범위 밖 행은 이미 지워지므로 날짜 규칙은 불필요하다.

`ok !== true`인 매장의 행은 **읽지도 쓰지도 않는다.** 삭제도, 신규 추가도 하지 않고 그대로 둔다. 정상 수집된 매장이 하나도 없으면 파일을 열기만 하고 저장 없이 종료한다.

엑셀은 "최근 1개월 현황"이며, 과거 리뷰가 지워지는 것은 의도된 동작이다.

```bash
python3 - "$HOME/skillwork/coupang.json" "$HOME/mnt/claude/쿠팡_저점수리뷰.xlsx" << 'ENDPY'
import json, sys, re, os
from datetime import date
import openpyxl
from openpyxl.styles import Alignment, Font, PatternFill
from openpyxl.styles.numbers import FORMAT_TEXT

DATA, XLSX = sys.argv[1], sys.argv[2]
HEADER = ['매장명','날짜','별점','주문번호','주문메뉴','리뷰내용']
WIDTHS = [24,14,10,16,40,60]

def star(n):
    try: n = int(n)
    except: return '?점'
    return '★'*n + '☆'*(5-n) if 1 <= n <= 5 else '?점'

def parse(s):
    m = re.match(r'(\d{4})-(\d{2})-(\d{2})$', str(s or '').strip())
    return date(int(m.group(1)), int(m.group(2)), int(m.group(3))) if m else None

with open(DATA, encoding='utf-8') as f:
    stores = json.load(f)

# ok가 정확히 True인 매장만 동기화 대상이다. 키가 없으면 실패로 본다(기본값 실패).
ok_stores = [s for s in stores if s.get('ok') is True]
skipped   = [s for s in stores if s.get('ok') is not True]
skip_names = {s['storeName'] for s in skipped}

if not ok_stores:
    print(json.dumps({'saved': False, 'reason': '정상 수집된 매장이 없어 저장하지 않음',
                      'skipped': [[s['storeName'], s.get('reason','')] for s in skipped]}, ensure_ascii=False))
    sys.exit(0)

try:
    wb = openpyxl.load_workbook(XLSX); print('[OK] 기존 엑셀 로드', file=sys.stderr)
except FileNotFoundError:
    wb = openpyxl.Workbook()
    if 'Sheet' in wb.sheetnames: del wb['Sheet']
    print('[OK] 새 엑셀 생성', file=sys.stderr)

if '전체' in wb.sheetnames:
    ws = wb['전체']
else:
    ws = wb.create_sheet('전체'); ws.append(HEADER)

if '삭제' in wb.sheetnames: wb.remove(wb['삭제'])   # 예전 버전이 만들던 시트 정리

rows = [list(r) for r in ws.iter_rows(min_row=2, values_only=True) if any(v not in (None,'') for v in r)]

current = {s['storeName']: {str(x['orderNo']).strip() for x in s.get('lowScore', [])} for s in ok_stores}

kept, deleted, untouched = [], 0, 0
for r in rows:
    name, no = str(r[0] or '').strip(), str(r[3] or '').strip()
    if name in skip_names:            # 확인 필요 매장 — 손대지 않는다
        kept.append(r); untouched += 1; continue
    # 날짜 열(주문일)로는 판정하지 않는다 — API 필터는 리뷰 작성일이라 주문일이 범위 밖이어도 수집된다.
    # 이번 수집에 없는 주문번호만 지운다. 범위 밖·삭제·4~5점 수정이 전부 여기 걸린다.
    missing = name in current and no not in current[name]
    if missing: deleted += 1
    else: kept.append(r)

existing = {(str(r[0]).strip(), str(r[3]).strip()) for r in kept}
new = []
for s in ok_stores:
    for x in s.get('lowScore', []):
        no = str(x.get('orderNo','')).strip()
        if not no: continue
        key = (s['storeName'], no)
        if key in existing: continue
        existing.add(key)
        new.append([s['storeName'], x.get('orderDate',''), star(x.get('stars')), no,
                    x.get('menu',''), x.get('reviewText','')])

data = kept + new
data.sort(key=lambda r: (parse(r[1]) or date(1970,1,1)))

# 셀 값만 None으로 비우면 빈 행이 남아 max_row가 계속 부풀고 파일이 커진다.
if ws.max_row >= 2:
    ws.delete_rows(2, ws.max_row - 1)

ws.row_dimensions[1].height = 20.0
for i, w in enumerate(WIDTHS, start=1):
    ws.column_dimensions[chr(64+i)].width = w
fill = PatternFill(patternType='solid', fgColor='2F5496')
for c in ws[1]:
    c.font = Font(name='Arial', bold=True, color='FFFFFF')
    c.fill = fill
    c.alignment = Alignment(horizontal='center', vertical='center', wrap_text=True)

align = Alignment(horizontal='center', vertical='center', wrap_text=True)
for i, row in enumerate(data, start=2):
    ws.row_dimensions[i].height = 40.0
    for j, v in enumerate(row, start=1):
        c = ws.cell(row=i, column=j)
        if j == 4:
            c.value = str(v); c.number_format = FORMAT_TEXT   # 주문번호는 텍스트 강제
        else:
            c.value = v
        c.alignment = align; c.font = Font(bold=False)

try:
    wb.save(XLSX); saved = XLSX
except Exception as e:
    # 세션별 홈($HOME/skillwork)은 세션이 끝나면 접근 불가라 사용자 폴더의 backup/ 에 둔다
    bdir = os.path.join(os.path.dirname(os.path.abspath(XLSX)), 'backup'); os.makedirs(bdir, exist_ok=True)
    saved = os.path.join(bdir, '쿠팡_저점수리뷰_저장실패_' + date.today().strftime('%Y%m%d') + '.xlsx')
    wb.save(saved)
    print(f'[WARN] 원본 저장 실패 ({e}) → {saved}', file=sys.stderr)

print(json.dumps({'saved': True, 'new': len(new), 'deleted': deleted, 'rows': len(data),
                  'untouched_rows': untouched, 'synced': [s['storeName'] for s in ok_stores],
                  'skipped': [[s['storeName'], s.get('reason','')] for s in skipped],
                  'saved_to': saved, 'new_rows': new}, ensure_ascii=False))
ENDPY
```

- `saved: false` → 저장하지 않았다. `skipped` 사유를 그대로 사용자에게 알리고 재실행을 안내한다.
- 저장 성공 → `[OK] 엑셀 저장 (신규 N건, 삭제 N건, 총 N행)`. `skipped`가 비어 있지 않으면 **반드시 함께 보고한다.**
- `saved_to`가 `backup/…저장실패…` 경로면 → 원본에 쓰지 못한 것이다. 엑셀을 닫고 다시 실행해달라고 알린다.

---

## Step 4: 정리

```
tabs_close_mcp(tabId=<탭ID>)
```

---

## 최종 보고 형식

```
조회 기간: YYYY-MM-DD ~ YYYY-MM-DD (최근 1개월)

| 매장 | 전체(API) | 수집 | 저점수 | 신규 | 별점분포(1~5) | 상태 |
|---|---|---|---|---|---|---|
| 김치찜의 정석 | N건 | N건 | N건 | N건 | .../... | 정상 |
| 참 제육 | ... | | | | | ⚠️ 확인필요: <reason> |
| 퍽퍽살이 싫어 내가 만든 곱도리 | ... |

엑셀: 신규 N건 추가, N건 삭제, 총 N행
```

- `전체(API)` 열은 `apiTotal`을, `수집` 열은 `collected`를 그대로 쓴다. 둘을 합쳐 쓰지 않는다 — 두 값이 벌어지는 것 자체가 신호다. `apiTotal`이 `null`이면 `확인필요`라고 적는다.
- `ok === false`인 매장은 상태 열에 `⚠️ 확인필요: <reason>`을 쓰고, **그 매장은 엑셀에 반영되지 않았음을 한 줄로 덧붙인다**("기존 행은 그대로 두었습니다").
- `ratingFail > 0`이면 표 아래에 `별점을 읽지 못한 리뷰 N건 — [별점확인필요] 표시로 저장됨`을 덧붙인다.
- `authExpired`가 있으면 재로그인 안내를 맨 위에 올린다.
- 신규 저점수 리뷰가 있으면 매장별로 내용을 나열한다. 없으면 "신규 저점수 리뷰 없음"이라고만 쓴다. 단 `ok === false`인 매장이 있으면 "없음"이라고 단정하지 말고 "확인 필요"라고 쓴다.

---

## 트러블슈팅

| 증상 | 원인 | 대응 |
|---|---|---|
| 결과 JSON이 끝에서 `[TRUNCATED]` 표식과 함께 끊김 | `javascript_tool` 반환값 약 1,000자 제한 | 그 결과는 버리고 매장 단위(그래도 넘으면 3건씩)로 나눠 읽는다. 전체를 한 번에 읽는 방식으로 되돌리지 말 것 |
| 한 매장만 저점수가 통째로 사라짐 | 실패를 모르고 빈 `lowScore`로 동기화함 | `ok` 플래그가 살아 있는지 확인. 저장 스크립트의 `ok_stores` 필터를 제거하지 말 것 |
| `apiTotal: null` (확인필요) | `total`/`totalCount`/`totalElements`가 전부 없음 = 응답 스키마 변경 | 로그의 응답 키 목록으로 `pick()` 후보 추가. **`\|\| 0`으로 되돌리지 말 것** — 5건만 긁고 끝난다 |
| `수집률 낮음 N/M` | 페이지 일부 실패 또는 중간 페이지 누락 | 재실행. 반복되면 `size`·필터 파라미터 확인 |
| `ratingFail`이 수집 건수와 같음 | `rating` 계열 필드명 변경 | 로그의 응답 키로 `pick()` 후보 추가. 그동안 리뷰는 `[별점확인필요]`로 보존된다 |
| `ratingFail`이 일부만 | 개별 리뷰의 별점 누락 | 정상 진행. 엑셀에서 `?점` + `[별점확인필요]`로 확인 |
| `STATUS:403` / 수집 중 403 | `size` 값을 바꿨을 가능성 (WAF 제약) | `size=5` 고정 확인. 403은 중단 조건이 아니다 |
| `STATUS:401` (Step 1) | 시작 시점 인증 만료 | 로그인 요청 후 Step 1 재시작 |
| `인증만료(401) — 수집 도중 세션 종료` | Step 2 진행 중 세션 만료 | 그 매장은 실패 처리되어 엑셀이 보존된다. 재로그인 후 재실행 |
| `API 오류 <code>: <message>` (HTTP 200이지만 `code`≠`SUCCESS`) | 서버가 거절한 요청. 실측: 없는 storeId → `10001 상점 정보를 찾을 수 없습니다`, `statusType` 누락 → `10007 입력 된 정보가 유효하지 않습니다`(틀린 값도 같을 것으로 추정, 미실측) | `error.message`를 그대로 읽는다. 10001이면 매장 정보 표의 storeId, 10007이면 `statusType`·날짜 파라미터를 확인 |
| `응답 구조 불명: [...]` | (1) `code` 필드가 없는 응답으로 스키마가 바뀜 (2) `code` 검사를 지나쳤는데 `content` 계열 키가 없음 | 사유에 찍힌 키 목록을 본다. `data`·`error`·`code`가 보이면 API 오류 봉투가 그대로 온 것이니 위 행으로. 그 외에는 로그의 응답 키 목록으로 `pick()` 후보 추가 |
| `apiTotal=0`인데 화면엔 리뷰가 보임 | 날짜 파라미터가 안 먹었거나 API 필터 기준이 바뀜 | 3개 매장 모두 0이면 특히 의심. `statusType` 파라미터가 빠진 경우는 0이 아니라 `API 오류 10007`(구버전이면 `응답 구조 불명: ["statusType"]`)로 나타난다(2026-09-09 실측) — 그 행을 볼 것 |
| 특정 매장만 `ok:false` | 그 매장 API 실패 | 나머지 매장은 계속 진행. 그 매장 엑셀 행은 보존됨. 재실행으로 복구 |
| 페이지 수가 매우 많아 오래 걸림 | 1개월 리뷰가 많은 매장 (size=5 고정) | 정상. 폴링 로그의 `p/pages`로 진행 확인 |
| 저장은 됐는데 파일이 계속 커짐 | 빈 행 누적 | `delete_rows`를 쓰는지 확인 |
| 날짜가 `날짜없음`으로 저장됨 | 날짜 필드명 변경 | `norm()`의 `pick()` 후보에 새 필드 추가 |