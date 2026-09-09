# coupang-review-skill

쿠팡이츠 사장님 포털에서 저점수(1~3점) 리뷰를 수집해 엑셀에 저장하는 Claude 스킬.

## 이 저장소의 역할

**기록 보관소다. 실행 경로가 아니다.**
스킬은 설치본(스킬 호출 시 표시되는 Base directory 아래 `SKILL.md` — 경로는 환경마다 다르다)을 읽고 돈다.
이 저장소는 그 사본과 점검 기록을 보관한다.

| 파일 | 무엇 |
|---|---|
| `SKILL.md` | 설치본 사본이자 정본. 수정은 여기서 하고 `.skill` 로 패키징한다 |
| `audit/checklist.md` | 정기 점검 절차 |
| `audit/last-audit.md` | 회차별 점검 기록 |

## 수정 흐름

저장소 `SKILL.md` 수정 → push → `.skill` 재패키징 → Claude 설정에서 재업로드.
**재업로드 전에는 실행에 반영되지 않는다.**

## 정기 점검

```
/coupang-review
정기 점검. 아직 고치지 말고 진단만. 뭘 고칠지는 내가 고를게.
저장소를 clone해서 audit/checklist.md 를 읽고 그대로 수행해라.
git clone https://github.com/LeeKwanBeom/coupang-review-skill
push 토큰: (여기에 붙여넣기)
```

## 자매 스킬

배민 쪽은 `LeeKwanBeom/baemin-review-skill` 에 같은 구조로 있다.
두 스킬은 목적이 같지만 수집 방식이 다르다 — 쿠팡은 same-origin API,
배민은 DOM 스크롤. 한쪽 방식을 다른 쪽에 이식하려 하지 말 것.
