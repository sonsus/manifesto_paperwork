2026-01-09
# 목표: 복합적인 요소가 들어있는 요청을 잘 분해해서 달성할 수 있는 계획을 세울 수 있는가? 엄격한 규칙과 순서를 요구하는 툴/API 환경에서? (정적인 계획 능력)

이 논문은 에이전트가 **"맨 땅에 헤딩하듯 하나씩 부딪히며(Trial and Error) 길을 찾는 능력"**보다는, **"처음부터 문제의 구조를 꿰뚫어 보고 완벽한 작업 지시서(Workflow)를 짜는 '기획 능력(Reasoning & Planning)'"**을 타겟팅했습니다. 

코딩 에이전트가 **복잡한 의존성(Dependency)**을 가진 여러 도구를 순서에 맞게 호출하는 Long-horizon Planning 능력은 정확히 타겟팅하고 있습니다.

하지만 실행 중간의 피드백을 받아 동적으로 경로를 수정하는 Adaptive Reasoning보다는, 사전에 정의된 도구 간의 입출력 관계(Type Matching)를 추론하는 Structural Reasoning에 더 가깝습니다.

> https://arxiv.org/pdf/2311.18760 에 나온 TaskBench는 처음에 instruction을 줬을 때 가용한 tool과 api들을 어떻게 활용하고 어떤 순서로 실행해야 요구사하에 맞출 수 있는지를 테스트하는 벤치인 것 같아. 


> 툴콜을 해서 파일을 찾았거나 못찾았거나 (상호작용) 에 따라서 앞으로 갈 길이 달라지거든. 그래서 내 생각에 TaskBench가 다루는게 상호작용 직전까지의 무언가처럼 느껴지기도 하거든? 대신 tool 들이나 api의 난이도가 상당해서 변별력있는 평가가 가능한... claude code를 사용할 때처럼 상호작용해가며 목표달성까지 동작하는 에이전트들 줄세우려고 만든 벤치마크도 존재해?

# 동적인 상호작용을 해가면서 문제를 해결하는 상황에 대한 벤치마크?
### 코딩 에이전트용 (레포 상호작용)

- **SWE-bench**: 실제 GitHub 이슈를 주고, 에이전트가 레포를 탐색/수정해 패치를 만든 뒤 테스트를 통과시키는지로 채점하는 실행 기반 벤치마크로 널리 쓰여.[](https://www.vals.ai/benchmarks/swebench)​
    
- **SWE-agent(+SWE-bench harness)**: SWE-bench를 대상으로, 에이전트가 컨테이너 환경에서 파일을 읽고/수정하고/테스트 돌리는 상호작용 루프를 수행하도록 만드는 대표 구현 중 하나야.[](https://swe-agent.com/0.7/usage/benchmarking/)​
    

### 웹/브라우저 상호작용 (멀티스텝 UI 액션)

- **WebArena**: e-commerce/포럼/협업툴/콘텐츠 관리 등 “실제 같은” 웹사이트가 들어있는 재현 가능한 환경에서, 고수준 지시를 구체적인 웹 UI 조작으로 수행해 과제가 끝났는지(기능적 정답)를 검증하는 벤치마크야.[](https://www.cmu.edu/flame/research/2024/webarena.html)​
    
- **BrowserGym**: Playwright 기반으로 브라우저 상호작용을 “gym 환경”처럼 표준화해서 여러 웹 에이전트 벤치를 같은 인터페이스로 평가할 수 있게 만든 프레임워크/에코시스템 쪽이야(특정 단일 벤치라기보다 ‘벤치들을 묶는 공통 러너’에 가까움).[](https://www.emergentmind.com/topics/browsergym-interface)​
    

### “컴퓨터 사용” 에이전트 (OS/앱까지)

- **OSWorld**: Ubuntu/Windows/macOS 같은 “실제 OS”에서 마우스/키보드 기반으로 여러 앱과 파일 I/O를 오가며 작업을 수행하고, 스크립트로 실행 기반 평가를 하는 벤치마크야(웹 + 데스크톱 워크플로우 포함).[](https://os-world.github.io/)​
### 범용 “LLM-as-Agent” 상호작용

- **AgentBench**: 여러 종류의 환경(멀티턴/오픈엔디드 포함)에서 LLM을 에이전트로 평가하는 벤치마크로 알려져 있어, 단순 툴 선택보다 “환경에서의 의사결정/상호작용” 성격이 강해.[](https://github.com/THUDM/AgentBench)​

# 그래서 실험 새로해야하나? ㄴㄴ Vending bench가 상호작용 끝판왕임.
