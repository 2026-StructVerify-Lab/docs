# StructVerify 리팩토링 계획 (backend / main)

> 작성: 2026-06 · 대상: `backend` 저장소 `main` 브랜치
> 이 문서는 **계획안** 이며, 작성 시점에 코드는 변경하지 않았습니다.
> 우선순위 표기: **P0(최우선) ~ P5**

---

## 1. 목적과 범위

`main` 브랜치 코드를 *동작은 유지하면서* 구조를 개선(refactor)하기 위한 계획입니다. 두 가지 동기가 있습니다.

1. **코드 품질** — 빠른 안정화(패치 누적)로 생긴 거대 파일·중복·죽은 코드를 정리한다.
2. **v4 토대** — "도메인 독립(BYO 데이터 소스/LLM)" 으로 가려면 엔진에 박힌 KOSIS·HCX 결합을 추상화 뒤로 밀어내야 한다.

범위는 결정사항(6절)에 따라 *코드 품질만* 또는 *v4 토대까지* 로 나뉩니다.

---

## 2. 진단 요약

읽기 전용 조사 결과(`structverify/` + `sv_platform/`, 약 30,685줄)입니다.

| 항목 | 측정값 | 의미 |
|---|---|---|
| 거대 파일 | 5개가 1,000줄 이상 | 단일 책임 위반, 테스트·온보딩 어려움 |
| 패치/버전 마커(`[vN]`/`[PNN]`/패치) | **344개 라인** | 빠른 패치 누적의 흔적(부채) |
| 죽은(주석처리) 코드 | **약 263줄** | 히스토리 보존 규칙으로 누적 |
| TODO/FIXME/HACK | 73개 | 미완 지점 |
| 엔진 코어의 `kosis` 언급 | **263회 · 32개 파일** | 데이터 소스 결합도 과다 |

### 거대 파일 (라인 수)

| 파일 | 라인 |
|---|---|
| `agent/loop.py` | 2,036 |
| `retrieval/kosis_source.py` | 1,567 |
| `detection/schema_inductor.py` | 1,491 |
| `retrieval/kosis_connector.py` | 1,309 |
| `agent/runtime_agent.py` | 1,185 |
| `agent/workspace.py` | 957 |
| `agent/tools/fetch_evidence.py` | 904 |
| `agent/tools/catalog_search.py` | 851 |
| `verification/verifier.py` | 818 |
| `agent/planner.py` | 816 |

### 중복 판정/계산 로직 (두 군데)

| `agent/loop.py` | `verification/verifier.py` |
|---|---|
| `_try_growth_rate_from_rows` | `_verify_growth_or_diff` |
| `_try_difference_from_rows` | `_find_best_match` |
| `_synthesize_verdict_from_observation` | `_verdict_from_error` |
| `_synthesize_verdict_from_calculate` | |

→ 증가율·차이 계산, 오차 구간 판정, 시점 매칭이 *두 모듈에 각각* 구현되어 있습니다.

---

## 3. 리팩토링 테마

### P0. KOSIS 결합도 제거 → DataSource 추상화  *(가치 최대, v4 토대)*

- **문제**: "도메인 독립" 을 표방하지만, KOSIS 전용 개념(`DT`/`PRD_DE`/`ITM_NM` 컬럼, `getMeta`, `objL`/`itmId` 차원, 통합검색 등)이 엔진 코어 32개 파일에 263회 박혀 있습니다. 이 상태로는 CSV/DB 같은 다른 정답 데이터 소스를 꽂을 수 없습니다.
- **방향**: KOSIS 전용 로직을 전부 `retrieval/` 의 *DataSource 구현 뒤로* 격리하고, 엔진 코어는 추상 계약(`StatRecord`, `Evidence`)만 알게 합니다.
- **산출물**: `BaseDataSource` 인터페이스 확정 → KOSIS를 그 한 구현으로 분리 → 이후 CSV/DB 추가가 "구현 하나 더" 가 됨.
- **난이도 高 / 가치 高.** v4(커스텀 데이터 소스)의 사실상 전제.

### P1. 중복 판정·계산 로직 통합  *(정확성/유지보수)*

- **문제**: 에이전트 루프(`loop.py`)가 결정론 판정을 *재구현* 해서 `verifier.py` 와 따로 돕니다. 증가율 부호 가드·오차 구간·시점 tier 매칭이 양쪽에 흩어져, 버그 수정 시 두 곳을 고쳐야 하고 회귀가 잦습니다.
- **방향**: 수치 비교·증가율/차이 계산·verdict 판정을 *하나의 verification 모듈* 로 모으고, `loop.py` 는 호출만 합니다. 단일 진실 소스(single source of truth).
- **난이도 中 / 가치 高.** 검증 정확도 회귀의 단골 원인 제거.

### P2. 거대 파일 분해

- `loop.py`(2,036) → ReAct 루프 본체 / verdict 합성(P1로 이동) / row 계산 헬퍼 / 가드 로 분리
- `kosis_source.py`·`kosis_connector.py` → row 매칭 / 차원 해석 / HTTP 호출 모듈로
- `schema_inductor.py`(1,491) → 프롬프트 / 파싱 / 폴백 분리
- `runtime_agent.py`(1,185, 내부 import 28개) → 오케스트레이션 책임 분리
- **난이도 中 / 가치 中.** 가독성·테스트성·온보딩 개선.

### P3. 죽은 코드 · 패치 마커 정리

- 주석처리 코드 약 263줄 + 패치/버전 마커 344개를 정리합니다.
- ⚠ **팀 규칙 충돌**: 협업 매뉴얼(`github-guide`)에 *"기존 코드 절대 삭제 금지, 주석 처리 후 버전 표기"* 가 있어서 이게 쌓였습니다. → **"리팩토링 시점엔 git 히스토리가 보존하니 죽은 주석 코드는 제거해도 된다"** 같은 *팀 합의* 가 먼저 필요합니다.
- **난이도 低 / 가치 中.** 합의만 되면 빠름.

### P4. LLM Provider 추상화  *(v4)*

- `utils/llm_client.py`(611)가 HCX 중심입니다. provider(hcx/openai/local) 인터페이스로 추상화합니다.
- Planner는 이미 `LLMCallable` 주입식이라 토대가 있으므로, *클라이언트 레이어만* 일반화하면 됩니다.
- **난이도 中 / 가치 高.** v4 BYO Key.

### P5. config · secret 일원화

- config 로딩이 여러 곳에 산재합니다. v4 BYO 제어판 스키마(`llm` / `data_sources` / `domain` / `advanced`)로 정리하고, 비밀값은 env로 일원화합니다.
- **난이도 中 / 가치 中.**

---

## 4. 권장 순서 & 안전장치

### 순서

```
P1(중복 통합) → P0(KOSIS 격리) → P2(분해) → P4(LLM 추상화) → P5(config) → P3(죽은코드, 합의 후)
```

- **P1을 먼저** 하는 이유: 판정 로직을 한 곳에 모아야, P0에서 KOSIS를 떼낼 때 안전합니다.
- P3(죽은 코드 정리)는 *팀 합의 후* 아무 때나 끼워 넣을 수 있습니다.

### 안전장치

1. **회귀셋을 안전망으로** — `eval/`(45 케이스, oracle/induce) + `tests/` 를 각 리팩토링 *전후로 실행* 해, verdict 분포가 바뀌지 않는지 확인합니다(behavior-preserving refactor).
2. **작은 PR 단위** — 거대 PR 금지. "한 파일 분해" / "한 중복 제거" 단위로 쪼갭니다.
3. **브랜치/커밋 컨벤션** — 팀 규칙대로 `refactor/v1/<이름>` 브랜치 + `refactor:` 커밋 타입.
4. **WIP 먼저 정리** — 현재 `main` 에 미커밋 변경(`README.md`·`sv_platform/config.py` 등)이 있으면, 커밋/스태시 후 *새 브랜치* 에서 리팩토링을 시작합니다.

---

## 5. 결정 필요 사항

| # | 결정 | 선택지 |
|---|---|---|
| (a) | 죽은 주석 코드 삭제 허용? | 삭제 금지 규칙 유지 ↔ *리팩토링 한정 예외* |
| (b) | 이번 리팩토링 범위 | *코드 품질만*(P1·P2·P3) ↔ *v4 토대까지*(P0·P4·P5 포함) |
| (c) | 회귀 기준 | eval 점수 *동일 유지* ↔ 구조 개선 우선(소폭 변동 허용) |

→ (a)·(b)·(c)가 정해지면 각 테마의 *상세 설계*(이동할 함수/모듈, 인터페이스 시그니처)로 내려갑니다.

---

## 6. 부록 — 다음 단계 후보

- **P0 상세 설계**: `BaseDataSource` 계약(메서드 시그니처) 확정 + KOSIS 격리 대상 함수 목록.
- **P1 상세 설계**: `loop.py` ↔ `verifier.py` 중복 함수 매핑표 + 통합 후 단일 모듈 구조.
- **P2 분해안**: 파일별 분리 단위(모듈 경계) 제안.

> 다시 강조: 이 문서는 **계획** 이며, 코드는 아무것도 변경하지 않았습니다.
