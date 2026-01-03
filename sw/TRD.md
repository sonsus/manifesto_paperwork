# TRD — Manifesto Core v1.0 Validation via LLM-BabyBench (v0.1)

> 목적: PRD의 요구사항을 구현 가능한 기술 요구로 구체화한다.
> 핵심: **core-v1(define*/executeAction/EffectRuntime/Patch/Apply/DAG/Explain)** 위에 BabyBench를 올리는 **어댑터 + 러너 + 평가 하네스**를 만든다.
> 모델은 의도적으로 **작은 모델(GPT-5-nano 또는 GPT-4o-mini)**를 사용한다.

---

## 0) 범위/철학 고정 (Non-negotiable)

* **계산:** Expression(DAG)만 사용 (외부 IO/시간/랜덤 금지)
* **실행:** Effect(Runtime)만 수행
* **변경:** Patch/Apply만 수행 (Truth 변화 단일 게이트)
* **missing access:** 예외 금지, `null`로 평가
* **배열 인덱스 path 금지:** `items.0.name` 같은 patch path는 v1.0에서 지원하지 않음
  → 배열은 `set('items', newItems)`처럼 **전체 교체**로만 변경

---

## 1) 시스템 아키텍처 (Module Map)

```
packages/
  core-v1/               # 이미 완성된 core (Sprint 1~4)
  babybench/
    src/
      adapter/           # paper format ↔ canonical JSON world
      world/             # world schema + helpers
      runtime/           # Effect handlers (env.step)
      policy/            # LLM policy (small model)
      runner/            # task loops (Predict/Plan)
      eval/              # metrics (Predict/Plan)
      prompts/           # templates + output schemas
      report/            # aggregation + exports
```

---

## 2) 데이터/환경 입력 요구 (Paper Integration)

### 2.1 입력 레코드 스키마(최소)

Predict row:

* `env_description: string`
* `initial_state: string`
* `action_sequence: string | string[]`
* `target_state: string`

Plan row:

* `env_description: string`
* `initial_state: string`
* `target_subgoal: string`
* `expert_action_sequence: string | string[]` (baseline/efficiency용)

Decompose는 v0.1 제외(옵션 v0.2).

### 2.2 데이터 소스

* HF dataset 또는 GitHub release 사용
* 최소 200~1,000 샘플로 sanity + 첫 비교 실험

---

## 3) Canonical World Schema (Core 친화)

### 3.1 설계 원칙

* **배열 인덱스 path 금지**를 고려해 내부 상태는 가능한 `Record` 기반
* availability 계산을 단순화하기 위해 **front-cell 요약 필드**를 포함(권장)

### 3.2 최소 스키마(예시, 구현 시 확정)

```ts
WorldState = {
  agent: { x: number; y: number; dir: 'N'|'E'|'S'|'W' },
  grid: {
    width: number; height: number,
    cells: Record<string /*c_x_y*/, Cell>
  },
  objects: Record<string /*objId*/, Obj>,
  inventory: string[],        // 단순화 가능
  front: {                    // 매 step 업데이트되는 요약(권장)
    blocked: boolean,
    canMove: boolean,
    canPickup: boolean,
    canToggle: boolean,
    targetId?: string | null
  },
  meta?: { mission?: string; subgoal?: string }
}
```

---

## 4) Manifesto State/Action 모델링 요구

### 4.1 State 구성

* `world` 모델 1개로 시작 (또는 `world + episode` 분리)
* `computed`는 Plan 목표 달성 여부 등 최소 세트만

### 4.2 Actions (6개 low-level)

* `turn_left`, `turn_right`, `forward`, `pickup`, `drop`, `toggle`

각 ActionNode는:

* `availability: Expression<boolean>`
* `effect: EffectSpec { type: 'env.step', params: { action: <token> } }`

Availability 기준(최소):

* turn_left/right: true
* forward: `world.front.canMove === true`
* pickup: `world.front.canPickup === true`
* drop: `inventory.length > 0` 또는 `carrying != null`
* toggle: `world.front.canToggle === true`

### 4.3 Effect Runtime 핸들러

* `env.step` handler는:

    * 입력: `ResolvedEffect.params.action`
    * 내부: step(state, action) 실행
    * 출력: Patch[]

v0.1 Patch 전략(단순):

* `set('world', nextWorld)` **전체 교체** 허용
* 이후 최적화(선택): 부분 patch

---

## 5) 환경 엔진(Transition) 요구

### 5.1 선택지

* A) **Python BabyAI/MiniGrid 호출** (권장: 초기)
* B) TS로 dynamics 구현 (v0.2+)

v0.1 권장:

* Python subprocess 또는 RPC(간단한 http)로 step 수행
* 반환: next state(JSON)
* 결정론 보장

### 5.2 계약

* step는 deterministic이어야 하며, 동일 입력이면 동일 next state 반환
* invalid action은:

    * availability가 false면 애초 실행하지 않음 (executeAction에서 UNAVAILABLE)
    * 만약 실행되면 runtime이 error로 처리 가능(테스트용)

---

## 6) Parser/Formatter(Structured text ↔ JSON) 요구

### 6.1 Parser

* `env_description`/`initial_state`의 structured format을 파싱하여 canonical JSON 생성
* 최소:

    * agent pose(x,y,dir)
    * object list(좌표/상태)
    * grid size

### 6.2 Formatter

* v0.1에서 Predict scoring을 위해:

    * canonical JSON → structured text 출력
* **exact match** 평가를 위해 whitespace/ordering 안정화 필요(정규화)

### 6.3 Robustness

* 파싱 실패 시:

    * 샘플 스킵 + 로그(또는 실패 집계)
* 형식 변동(공백/줄바꿈)을 tolerable하게 처리

---

## 7) Policy(LLM Agent) 요구 — Small Model

### 7.1 정책 형태

v0.1은 **step-policy**로 고정:

* 매 스텝 action 1개 선택
* 입력: state(Structured), 목표/서브골(Plan), available actions
* 출력: 단일 action token

### 7.2 출력 스키마

* 반드시 enum 6개 중 1개만 출력
* 파싱 실패 시 1회 재질문(repair) 허용

### 7.3 Prompt 템플릿

* baseline prompt(naive) vs manifesto prompt(availability 포함) 분리
* ToT는 v0.1에서 optional(비교 통제)

---

## 8) Runner(실행 루프) 요구

### 8.1 Predict Runner

* 입력: (initial_state, action_sequence)
* 루프:

    * 각 action을 executeAction으로 실행
* 최종 state를 formatter로 stringify
* 평가: exact match + L1 distance(가능하면 agent pose 파싱)

### 8.2 Plan Runner

* 입력: (initial_state, target_subgoal)
* 루프:

    * state → policy(action)
    * executeAction(action)
    * goal satisfied면 종료
    * 최대 step limit(예: 128) 설정
* 평가:

    * success 여부(환경이 목표 달성)
    * efficiency ratio(옵션: expert_action_sequence length 사용)

### 8.3 Instrumentation

* invalid action rate(availability false로 인한 UNAVAILABLE 비율)
* first failure step
* trace(opt-in) 저장

---

## 9) Evaluation/Scoring 요구

### 9.1 Predict

* exact match(정규화된 문자열 기준)
* L1 distance(에이전트 위치 추출 가능 시)

### 9.2 Plan

* success rate: 실행 결과가 목표 달성
* efficiency ratio: `len(expert_opt)/len(llm_actions)` (가능 시)

---

## 10) Explain/Debug 요구

### 10.1 Explain Availability

* UNAVAILABLE 발생 시 explainAvailability(action, snapshot) 실행
* 이유 path 리스트 + 요약 문장 저장

### 10.2 Failure Taxonomy

* UNAVAILABLE: availability false (계산 결과)
* EXECUTION_FAILED: runtime error
* APPLY_FAILED: invalid patch path
* EVALUATION_FAILED: DAG recompute 실패

---

## 11) 테스트 요구 (Minimal)

* adapter parser/formatter round-trip 테스트
* env.step handler:

    * 동일 입력 → 동일 출력(determinism)
* executeAction:

    * availability false → UNAVAILABLE
    * effect→patch→apply→recompute 루프 정상
* scoring:

    * Predict exact match normalization 테스트

---

## 12) MVP 산출물(Deliverables)

* `babybench` 패키지

    * adapter(parser/formatter)
    * env runtime(handler)
    * policy(step LLM prompt + parser)
    * runners(Predict/Plan)
    * eval(metrics)
    * report(summary csv/json)
* baseline vs manifesto 비교 리포트(수치 + 실패 explain 사례)

---

## 13) 리스크 & 완화

* 포맷 brittle → canonical JSON + stable formatter + normalization
* 작은 모델 format 오류 → strict enum + 1회 repair
* Python 의존 → v0.1에서는 허용, v0.2에서 TS port
* efficiency(optimal) 계산 어려움 → v0.1은 success 중심, 효율은 옵션

---

## 14) 마일스톤(Implementation Order)

1. Adapter(parser/formatter) + Predict runner (sanity)
2. env.step runtime(Python 연동) + executeAction loop
3. Plan runner + small LLM policy
4. Baseline vs Manifesto 비교 실험
5. Explain 기반 failure 분석 리포트

---

*End of TRD v0.1*
