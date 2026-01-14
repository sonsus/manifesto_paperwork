# FDR — Manifesto Core v1.0 Validation via LLM-BabyBench (v0.1)

[← README](README.md)

## 0. 문서 목적

본 문서는 **Manifesto Core v1.0을 LLM-BabyBench로 실증하기 위한 근본 설계 결정(FDR)**을 기록한다.
여기서의 결정은 **실험 해석·비교·성공 기준을 고정**하며, 이후 SPEC/TRD는 본 문서를 **단일 근거(Source of Truth)**로 따른다.

> 핵심 질문:
> **“작은 LLM이라도, 올바른 Runtime(Core)을 얹으면 성과가 개선되는가?”**

---

## 1. 실증의 핵심 가설 (Primary Hypothesis)

> **H:** 잘 설계된 Runtime(Manifesto Core)이 제공하는
> *결정론적 상태 관리, 가용성(availability) 가이딩, 패치 기반 변경, 구조적 설명*은
> **작은 LLM(GPT-5-nano 또는 GPT-4o-mini)**에서도
> BabyBench의 Plan/Reasoning 성능을 유의미하게 개선한다.

이 실증의 목적은 **모델 지능의 비교가 아니라 구조의 비교**다.

---

## 2. Agent의 정의 (FDR-B1)

### 결정

> **Agent = LLM + Manifesto Runtime(Core)**

### 근거

* LLM-BabyBench는 실행 검증을 전제로 하며, LLM은 **정책(policy)** 역할에 가깝다.
* Manifesto의 가치는 **정책을 둘러싼 Runtime의 안정성/결정론/설명 가능성**에 있다.

### 결과

* 실험에서 “Agent 성능”은 **LLM 단독이 아니라 Runtime 포함 성능**을 의미한다.
* “LLM이 멍청하다”는 비판은 본 실증의 범위를 벗어난다.

---

## 3. 비교 기준의 고정 (FDR-B2)

### 결정

> **주요 비교는 동일 모델에서의 ‘Runtime 유무’ 비교다.**

### 근거

* 본 실증의 메시지는 **“모델을 바꾸지 않고 구조를 바꾸면 달라진다”**다.
* 논문 상위 모델(Claude/Qwen)은 **맥락(reference)**으로만 사용한다.

### 결과

* 헤드라인 비교:
  **GPT-4o-mini (naive) vs GPT-4o-mini (Manifesto Runtime)**
* 논문 수치는 **상대 위치 확인** 용도로만 사용한다.

---

## 4. 태스크 우선순위 (FDR-B3)

### 결정

> **v0.1의 핵심 성과는 Plan task에서 정의한다.**

### 근거

* Predict는 포맷/파서 품질에 크게 의존하며 Runtime의 가치가 덜 드러난다.
* Decompose는 OmniBot 의존성이 커서 v0.1 범위를 넘는다.
* Plan은 **가용성, 장기 안정성, 실행 검증**이 모두 드러나는 핵심 태스크다.

### 결과

* **Predict:** 하네스 검증/정합성 확인용
* **Plan:** 주 실증 지표
* **Decompose:** v0.2 이후 확장

---

## 5. Availability 위반의 의미 (FDR-B4)

### 결정

> **Availability false는 실패가 아니라 진단 신호다.**

### 근거

* Manifesto에서 availability는 **계산 결과**다.
* 불가능 행동은 Runtime이 막아야 하며, 그 이유는 설명 가능해야 한다.

### 결과

* invalid action은:

    * 실행 실패 ❌
    * **진단 지표 ⭕**
* 수집 지표:

    * invalid action rate
    * first invalid step index
    * explain(why unavailable)

---

## 6. Environment Step의 위치 (FDR-B5)

### 결정

> **BabyBench 환경 전이는 Effect(Runtime)로 모델링한다.**

### 근거

* 환경은 Core 외부 세계다.
* 결정론적이더라도 **계산(DAG)** 안으로 들이지 않는다.

### 결과

* `env.step`은 Effect handler로만 존재한다.
* Predict/Plan 모두 동일한 Effect 경계를 사용한다.

---

## 7. 성과 증명의 우선 지표 (FDR-B6)

### 결정

> **v0.1에서는 안정성 지표를 1차 성과로, 성공 지표를 2차 성과로 본다.**

### 근거

* 작은 모델의 절대 성공률에는 상한이 있다.
* Runtime의 가치는:

    * 불가능 행동 감소
    * 실패 지연
    * 실패 원인 설명
      에서 먼저 드러난다.

### 1차 지표

* invalid action rate
* first failure step (mean/median)
* availability explain coverage

### 2차 지표

* Plan success rate
* Efficiency ratio(가능 시)

---

## 8. Prompting의 위치 (Control Variable)

### 결정

> **Prompting은 통제 변수이며, Runtime 효과와 분리한다.**

### 근거

* ToT/CoT는 search 효과를 섞는다.
* v0.1의 목적은 **Runtime 구조 효과 분리**다.

### 결과

* 기본: 단순 step-policy prompt
* ToT는 **부가 실험**으로만 보고한다.

---

## 9. v0.1 범위에서 의도적으로 하지 않는 것

* array patch ops
* parent→child DAG 확장
* partial observability
* 고급 search prompting
* 모델 간 직접 랭킹 경쟁

이는 모두 **v0.2+ 확장**으로 유예한다.

---

## 10. FDR 요약 (결정 표)

| ID | 주제           | 결정                |
| -- | ------------ | ----------------- |
| B1 | Agent 정의     | LLM + Runtime     |
| B2 | 비교 기준        | 동일 모델, Runtime 유무 |
| B3 | 핵심 태스크       | Plan              |
| B4 | Availability | 실패 ❌ / 신호 ⭕       |
| B5 | env.step     | Effect            |
| B6 | 성과 지표        | 안정성 → 성공          |
| B7 | Prompting    | 통제 변수             |

---

## 11. 종료 선언

본 FDR은 **BabyBench 실증 v0.1의 모든 해석 기준을 고정**한다.
이 문서가 합의되면:

* PRD/TRD/SPEC은 본 결정을 따른다.
* 실험 결과 해석은 본 기준으로만 수행한다.
* 논쟁은 “구현/데이터”에 대해서만 허용된다.

> **이 실증은 모델 경쟁이 아니라, Runtime의 증명이다.**

---

*End of FDR — BabyBench Validation v0.1*
