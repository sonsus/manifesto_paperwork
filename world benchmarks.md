# 2025-12-28
## alfworld, textworld vs. vending bench, baby bench

> alfworld / textworld가 NLP커뮤니티 첫 반응이었다.

gemini: alfworld/textworld, vending bench, baby bench의 공통점과 차이점

- **AlfWorld/TextWorld (1세대/기본기):** "복잡한 지시를 이해하고, 탐험하여 **과제를 해결**할 수 있는가?" (문제 해결력 중심)
	- **할 수 있냐?**: multi-hop reasoning, exploration under OOD env
    
- **Vending/BabyBench (2세대/진단):** "장시간 동안 **정신줄을 놓지 않고(Coherence)** 일관성을 유지하거나, **인과관계(Grounding)**를 진짜로 이해하는가?" (안정성 및 기초 논리 중심)
	- **정밀진단**: 얼마나 오랫동안 얼마나 일관성있게 성능을 유지할까? 

| **특성**           | **AlfWorld / TextWorld**       | **Vending Bench / LLM-BabyBench**               |
| ---------------- | ------------------------------ | ----------------------------------------------- |
| **주요 목적**        | **Task Completion** (과제 완수 여부) | **Diagnostics** (모델의 특정 약점 진단)                  |
| **평가 초점**        | 복잡한 지시 이해, 환경 탐색, 시각-언어 연결     | **장기 기억 유지(Vending)**, **물리적 인과 이해(Baby)**      |
| **환경 복잡도**       | 높음 (다양한 사물, 방대한 상호작용)          | 낮음~중간 (단순한 규칙, 격자 세계, 자판기)                      |
| **Time Horizon** | 단기~중기 (하나의 에피소드 해결)            | **초장기(Vending)** 또는 정밀한 단기(Baby)                |
| **실패 요인**        | 탐색 실패, 비전 인식 오류, 계획 수립 불가      | **문맥 손실(Context Drift)**, **환각(Hallucination)** |
| **대표 질문**        | "이 낯선 부엌에서 사과를 찾아 씻어올 수 있어?"   | "1년 내내 자판기 재고 안 까먹고 관리할 수 있어?"                  |
## 기 수행벤치
- [x] ## LLM babybench (현재 수행 벤치)
	- https://www.themoonlight.io/paper/share/6e76b3ab-a372-4e59-90c1-f5054bf6bfa0 
- [x]  Vending bench (현재 수행 벤치)

## 미탐험 벤치 (Related works 방향잡기)

> 사람들이 무엇을 궁금해하는가? 

포화되지 않았지만 world model 상호작용을 다루고 있는 유명한 벤치 찾아주기
- agentgym, https://aclanthology.org/2025.acl-long.1355/
- 벤치 이름은 없지만 우리 웍에 related로 넣을만함 https://arxiv.org/abs/2411.08794
- tools and sandbox https://aclanthology.org/2025.findings-naacl.65/
- MCP-RADAR https://arxiv.org/abs/2505.16700