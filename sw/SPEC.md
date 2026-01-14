# SPEC — Manifesto Core v1.0 Validation via LLM-BabyBench (v0.1)

[← README](README.md)

> 본 SPEC은 **BabyBench 실증(v0.1)**을 위한 구현 계약을 고정한다.
> 상위 근거 문서: **FDR — BabyBench Validation v0.1**
> 목표: **작은 모델(GPT-5-nano 또는 GPT-4o-mini)** + Manifesto Runtime(Core)으로 **Plan 성능/안정성 개선**을 실증한다.

---

## 0. 헌법/범위 고정 (MUST)

### 0.1 Core 헌법 준수

* 계산은 **Expression(DAG)** 으로만 (MUST)
* 실행은 **Effect(Runtime)** 로만 (MUST)
* 변경은 **Patch/Apply** 로만 (MUST)
* missing access는 예외가 아니라 `null` (MUST)
* 배열 인덱스 path(`items.0.name`) Patch는 v0.1에서 금지 (MUST NOT)

### 0.2 실증 FDR 준수

* Agent = **LLM + Runtime(Core)** (MUST)
* 비교는 동일 모델의 **Runtime 유무** (MUST)
* v0.1의 핵심 성과는 **Plan** (MUST)
* availability false는 실패가 아니라 **진단 신호** (MUST)
* env.step은 Effect로만 모델링 (MUST)
* 성과는 안정성 지표 우선 (SHOULD)

---

## 1. 패키지 구조 (SHOULD)

```
packages/babybench/
  src/
    adapter/       # parsing/formatting
    world/         # canonical world schema
    actions/       # ActionNode definitions
    runtime/       # env.step effect handler + runtime bootstrap
    policy/        # LLM policy (small model)
    runner/        # Predict/Plan loops
    eval/          # metrics
    explain/       # explain extraction helpers
    report/        # aggregation + export
  tests/
```

---

## 2. 데이터 입력 계약 (Dataset I/O)

### 2.1 Record Types

**PredictRow**

* `env_description: string`
* `initial_state: string`
* `action_sequence: string | string[]`
* `target_state: string`

**PlanRow**

* `env_description: string`
* `initial_state: string`
* `target_subgoal: string`
* `expert_action_sequence?: string | string[]` (efficiency optional)

**DecomposeRow**는 v0.1 제외 (MAY include later).

### 2.2 Dataset Loader

* HF dataset 또는 로컬 jsonl/csv를 로드할 수 있어야 함 (MUST)
* 최소 subset slicing 지원 (start/end, count) (SHOULD)

---

## 3. Canonical World Schema (JSON)

### 3.1 원칙

* 내부 상태는 **Record 중심**으로 구성하여 배열 인덱스 path 요구를 제거한다. (MUST)
* availability 계산을 단순화하기 위해 **front 요약 필드**를 포함한다. (SHOULD)
* 월드 데이터는 `snapshot.data.world` 아래에 존재한다. (MUST)

### 3.2 최소 스키마 (MUST)

```ts
type Dir = 'N' | 'E' | 'S' | 'W'

type WorldState = {
  agent: { x: number; y: number; dir: Dir }

  grid: {
    width: number
    height: number
    cells: Record<string /*c_x_y*/, Cell>
  }

  objects: Record<string /*objId*/, Obj>

  inventory: string[]  // 단순화 허용 (v0.1)

  front: {
    blocked: boolean
    canMove: boolean
    canPickup: boolean
    canToggle: boolean
    targetId: string | null
  }

  goal?: {
    type: 'subgoal' | 'mission'
    text: string
  }

  meta?: Record<string, unknown>
}
```

* `Cell`, `Obj` 필드의 세부는 파서 구현 범위에서 최소화 가능 (MAY).
* 단, **agent pose / front flags**는 Plan에서 필수. (MUST)

---

## 4. Parsing & Formatting (Structured Text ↔ JSON)

### 4.1 Parser (MUST)

* 입력: `env_description`, `initial_state` (paper structured format)
* 출력: `WorldState` JSON
* 파싱 실패 시:

    * v0.1: 샘플 스킵 + 에러 로깅 (SHOULD)
    * 전체 러너를 크래시시키지 않는다 (MUST NOT)

### 4.2 Formatter (Predict용, MUST for Predict)

* 입력: `WorldState`
* 출력: paper structured style 텍스트
* 정규화 규칙:

    * object ordering, whitespace, line breaks를 안정화 (SHOULD)

---

## 5. Action Vocabulary & Output Contract

### 5.1 Low-level Actions (MUST)

다음 6개 토큰만 허용:

* `turn_left`
* `turn_right`
* `forward`
* `pickup`
* `drop`
* `toggle`

### 5.2 Policy Output (MUST)

* Step-policy 모드: 한 step에 **토큰 1개**만 출력
* 파싱 실패 시: 1회 repair 재질문 허용 (MAY)

---

## 6. Manifesto State/Action Modeling

### 6.1 StateNode (MUST)

* `defineState({ world: WorldModel, ... })` 형태로 구성한다.
* v0.1 최소: `world` 모델 1개만으로 시작 가능.

### 6.2 WorldModel (MUST)

* `schema`는 `WorldState`를 포함해야 함.
* computed는 최소 세트만 허용 (SHOULD):

    * `goalSatisfied` (Plan termination)
    * `isStuck` (optional)
    * `stepCount` (optional)

### 6.3 ActionNodes (MUST)

각 action은 아래를 포함:

* `availability: Expression<boolean>`
* `effect: EffectSpec { type: 'env.step', params: { action: <token> } }`

Availability 최소 규칙 (MUST):

* `turn_left/right`: `true`
* `forward`: `world.front.canMove == true`
* `pickup`: `world.front.canPickup == true`
* `drop`: `inventory.length > 0` (또는 carrying != null, 구현 선택)
* `toggle`: `world.front.canToggle == true`

### 6.4 Availability Violation Handling (MUST)

* availability false는:

    * action 실행 실패가 아니라 **UNAVAILABLE** 결과로 기록
    * invalid action rate 등 진단 지표로 집계

---

## 7. Environment Step as Effect (env.step)

### 7.1 Effect Type (MUST)

* `env.step`만 필수 제공(v0.1)

### 7.2 Effect Handler Contract (MUST)

* 입력: `ResolvedEffect.params.action` (토큰 1개)
* 실행: (WorldState, action) → nextWorldState
* 출력: Patch[]

### 7.3 Patch Strategy (v0.1, MUST)

* v0.1에서는 간단히 전체 교체 허용:

    * `set('world', nextWorld)` 1개 patch로 반환 가능 (MUST)
* 부분 patch 최적화는 v0.2+로 유예 (MAY)

### 7.4 Implementation Option (SHOULD)

* 초기 구현은 Python BabyAI/MiniGrid 호출을 권장 (SHOULD):

    * 입력 JSON → 출력 JSON
    * deterministic 보장
* TS 구현은 v0.2+ (MAY)

---

## 8. Runner Contracts

### 8.1 Predict Runner (MUST)

입력:

* PredictRow

절차:

1. parse initial_state → snapshot 생성
2. action_sequence를 순차 실행:

    * 각 step: `executeAction(state, snapshot, actionNode, runtime)`
3. 최종 snapshot.data.world을 format하여 state text 생성
4. scoring 수행

출력:

* `PredictResult { success, exactMatch, l1Distance?, traceSummary }`

### 8.2 Plan Runner (MUST)

입력:

* PlanRow

절차:

1. parse initial_state → snapshot 생성
2. 반복:

    * policy가 action token 1개 선택
    * executeAction 수행
    * goalSatisfied면 종료
    * maxSteps(기본 128) 초과 시 실패
3. scoring 수행

출력:

* `PlanResult { success, steps, invalidActionRate, firstInvalidStep?, efficiency?, traces }`

### 8.3 Baselines (MUST)

* Baseline A: naive prompt (availability 없이) step-policy
* Baseline B: manifesto step-policy (availability 포함)
* **동일 모델**로 비교 (MUST)

---

## 9. Scoring & Metrics (v0.1)

### 9.1 Predict (MUST)

* success rate: exact match
* L1 distance: agent pose 추출 가능 시 계산 (SHOULD)

### 9.2 Plan (MUST)

* success rate: goalSatisfied 달성 여부
* invalid action rate: UNAVAILABLE 비율 (MUST)
* first invalid step index (SHOULD)
* efficiency ratio: expert length 기반(옵션) (MAY)

---

## 10. Explain & Debug

### 10.1 Explain Availability (MUST)

* UNAVAILABLE 발생 시 `explainAvailability(action, snapshot)` 실행 가능해야 함
* 산출:

    * unavailability reasons(paths)
    * human-readable summary(옵션)

### 10.2 Failure Taxonomy (MUST)

* UNAVAILABLE (availability false)
* EFFECT_RESOLUTION_FAILED
* EFFECT_EXECUTION_FAILED
* APPLY_FAILED
* EVALUATION_FAILED

---

## 11. Logging/Reporting Outputs

### 11.1 Result Export (SHOULD)

* JSONL: per episode 결과 + 주요 메타(steps, invalid rate)
* CSV: aggregated metrics per difficulty bucket (가능 시)

### 11.2 Experiment Report (MUST)

* 동일 모델 대비(Runtime vs naive) 비교 표
* 최소 10개 실패 케이스 explain 예시 포함

---

## 12. Tests (MUST)

최소 테스트 요구:

* parser: representative samples parse 성공
* formatter: round-trip stability(부분)
* env.step: determinism test
* executeAction: end-to-end 1 step + multi-step
* Plan loop: invalid action rate 계산
* Explain: UNAVAILABLE 케이스에서 reason path 반환

---

## 13. Acceptance / Exit Criteria (v0.1)

v0.1 완료 조건:

1. Predict runner가 end-to-end 동작 (MUST)
2. Plan runner가 end-to-end 동작 (MUST)
3. 작은 모델로 baseline vs manifesto 비교 결과가 생성됨 (MUST)
4. invalid action rate 및 explain examples가 리포트에 포함됨 (MUST)

---

*End of SPEC — BabyBench Validation v0.1*
