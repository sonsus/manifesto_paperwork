# PRD — Manifesto Core v1.0 Validation via LLM-BabyBench (v0.1)

## 1) 목적 (Purpose)

Manifesto Core v1.0의 런타임(계산/실행/변경/설명)이 **실제 벤치마크 환경에서 성능을 끌어올릴 수 있는지**를 실증한다.
특히 “똑똑한 모델”이 아니라 **작은 모델(GPT-5-nano 또는 GPT-4o-mini)**로도 성과를 낼 수 있음을 보여주기 위해, LLM-BabyBench를 첫 실증 타깃으로 선택한다.

핵심 가설:

> **잘 설계된 Runtime(Core) + 결정론적 상태/가용성/패치/설명 루프가 있으면,
> 작은 LLM도 Plan/Reasoning 성능을 유의미하게 끌어올릴 수 있다.**

---

## 2) 성공 정의 (Success Criteria)

### S0. 하네스 정합성(필수)

* 논문/공식 하네스의 평가 절차(실행 검증, success 판단)를 **동일하게 재현**
* 최소한 Predict/Plan에서 baseline 재현(수치 근사) 또는 일관된 상대 순위 확인

### S1. “런타임 효과”를 작은 모델에서도 증명

* **GPT-4o-mini 또는 GPT-5-nano**로 Plan 성공률/효율(또는 Decompose 지표)에서

    * **단순 프롬프트(naive) 대비** Manifesto Runtime 기반 agent가 유의미한 개선
* “모델 바꾸지 않고 구조만 바꿔서 개선”을 보여주는 비교표 제공

### S2. 설명 가능성(Explain) 제공

* 실패 케이스에서 “왜 불가능했는지/왜 실패했는지”를 Explain Graph 기반으로 재구성
* 단순 로그가 아니라 **구조적 근거 트리**로 제공

---

## 3) 타깃 범위 (Scope)

### 3.1 포함(Phase)

* **Phase 1: Predict**

    * 하네스 검증 및 parser/formatter 안정화
* **Phase 2: Plan**

    * Manifesto Runtime의 핵심 가치(availability + step execution + patch/apply + explain) 검증
* **Phase 3: Decompose (선택/확장)**

    * OmniBot executor 연동까지 포함되므로 후순위
    * PR/CR/ACI는 v0.2+로 유예 가능

### 3.2 제외(Non-Goals)

* partial observability variant(논문 외 확장)
* 고급 ToT/검색 프롬프팅 최적화(구조 효과를 먼저 보기 위함)
* v1.1 기능(Parent-path 확장, array patch ops 등)

---

## 4) 사용자/가치 (User & Value)

### U1. Manifesto Core 사용자(개발자)

* “이 Core를 쓰면 뭐가 좋아지나?”를 벤치마크로 증명

### U2. Agent 연구자

* 모델이 아니라 **런타임 구조**로 성능이 개선될 수 있음을 보여주는 사례 제공

---

## 5) 제품 가설 (Hypotheses)

H1. **Availability 가이딩**(불가능 행동 차단 + 이유 제공)만으로도 작은 모델의 Plan 성공률이 상승한다.
H2. **Patch/Apply 기반의 결정론적 상태 갱신**이 long-horizon에서 오류 누적을 줄인다.
H3. **Explain Graph**가 실패 분석을 자동화해 prompt/정책 개선 루프를 빠르게 만든다.
H4. Predict에서 **정확한 state parsing/serialization**이 Plan 성능의 상한을 결정한다.

---

## 6) 시스템 구성(높은 수준 아키텍처)

### 6.1 구성 요소

1. **BabyBench Adapter**

* Dataset row 로딩(HF)
* `env_description` + `initial_state` 파싱
* canonical `WorldState(JSON)`로 변환

2. **Manifesto World Model (State)**

* `world.*` 데이터 필드
* `world.front.*` 등 availability 계산을 단순화하는 “요약 필드” 포함(권장)

3. **Actions (6개 저수준)**

* left/right/forward/pickup/drop/toggle
* `availability: Expression<boolean>`
* `effect: EffectSpec('env.step', { action })`

4. **EffectRuntime Handler**

* `env.step` 실행
* 외부 simulator(Python BabyAI/MiniGrid) 또는 TS 구현 호출
* 결과를 **Patch[]**로 반환(초기에는 `set('world', nextWorld)`로 전체 교체 가능)

5. **executeAction 파이프라인**

* availability → resolveEffect → runtime.execute → applyPatches → DAG recompute

6. **LLM Policy (Small Model)**

* Step policy: 매 step 한 개 action 선택
* 입력: structured state + mission/subgoal + available actions(+ optional explain hints)
* 출력: action token(단일)

7. **Scorer**

* Predict: exact match + L1 distance
* Plan: simulator success + efficiency ratio(옵션)
* Decompose: PR/CR/ACI(후순위)

---

## 7) 핵심 기능 요구사항 (Functional Requirements)

### FR1. State Parser/Formatter

* 논문 Structured format과 호환되는 입력 파서
* canonical JSON state로 round-trip 가능
* serializer(예측 state 출력용) 구현

### FR2. Environment Step Engine

* 입력: (state, action) → next state
* deterministic
* 에러/불가능 행동 처리 규칙 명확화(availability 위반 시 fail/skip)

### FR3. Action Availability Expressions

* forward/pickup/toggle 등의 precondition을 표현식으로 정의
* true 외 값은 false 처리(코어 규칙 준수)

### FR4. EffectRuntime 통합

* `env.step` handler 구현
* patch 반환 + applyPatches + changedPaths 루프 연결

### FR5. LLM Agent 정책(작은 모델)

* 기본: greedy step selection
* 출력 포맷 엄격(액션 토큰)
* 포맷 오류 처리(재질문/클린업) 최소 로직

### FR6. Evaluation Harness 재현

* Predict/Plan 평가 루프 구현
* 논문 metrics 산출:

    * Predict: success, L1
    * Plan: success, efficiency(가능 시)

### FR7. Explain 기반 디버깅

* 실패 케이스에서 availability explain
* invalid action 시점/원인 기록

---

## 8) 실험 설계 (Experiment Design)

### 8.1 비교군(컨트롤)

* Baseline A: naive prompt(상태 + 목표)로 action sequence 직접 생성
* Baseline B: naive step policy(availability 무시 또는 단순 마스킹만)

### 8.2 실험군(Manifesto)

* Experiment M: Manifesto Core loop + availability gating + patch/apply + explain 기반
* 모델: **GPT-4o-mini 또는 GPT-5-nano**

### 8.3 지표

* Predict: exact match, L1
* Plan: success rate, efficiency ratio, invalid action rate(추가)
* Explain: 실패 원인 분류 가능성(정성/정량)

---

## 9) 마일스톤 (Milestones)

### M0. Repo/Runner Skeleton

* babybench runner 폴더/CLI
* dataset loader

### M1. Predict Harness

* state parse/serialize 완성
* predict score 계산
* 몇 개 샘플로 sanity pass

### M2. Plan Step Engine + Actions

* env.step handler 연결
* 6 actions + availability expressions
* executeAction 루프 완주

### M3. Plan Evaluation + Baselines

* baseline vs manifesto 비교
* small 모델로 수치 확보

### M4. Explain 기반 분석 리포트

* 실패 케이스 Top N에 대한 explain 출력
* “왜 작은 모델도 개선되는지” 서술 가능하게

### M5. (선택) Decompose 확장

* OmniBot-like executor 붙이기
* PR/CR/ACI 계산

---

## 10) 리스크 및 완화 (Risks)

* R1: state parsing brittleness(포맷 에러)

    * Mitigation: canonical JSON으로 변환 후 내부 처리, 외부 출력만 포맷 맞추기
* R2: Plan 환경 구현 비용

    * Mitigation: 초기에는 Python simulator 호출로 단축
* R3: 작은 모델의 format 실패

    * Mitigation: action token 단일 출력 강제 + strict validator + 1회 재질문
* R4: efficiency ratio 계산(OmniBot optimal 필요)

    * Mitigation: v0.1에서는 success 중심, efficiency는 옵션

---

## 11) 산출물 (Deliverables)

* `babybench-adapter/` (parser/formatter)
* `env-step-runtime/` (Effect handler)
* `agent-policy/` (small model step selector)
* `runner/cli` (Predict/Plan 실행)
* `report/` (baseline vs manifesto 비교 + explain 사례)

---

## 12) v0.1 완료 조건(Exit Criteria)

* Predict/Plan 모두에서 end-to-end 실행 가능
* small model로 baseline 대비 개선이 관측됨(최소 1개 난이도 구간)
* explain 출력으로 실패 원인이 구조적으로 제시됨
