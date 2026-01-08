# 2025-12-28
- Parallel world (or World-model) exploration 을 통해 최적의 World model을 찾고
- 이를 통해 LLM Agent의 non-determinism으로부터 오는 수 많은 문제점들을 효과적으로 해결하는 방법이다.
	- 문제 1: long-context dilution of memory and intent.
	- 문제 2: reasoning observability and controllability.
	- 위 두 문제 모두 agent 관점에서 효과적, 효율적 탐색을 위해 매우 중요하다고 생각됨[^1]
- 그리고 world exploration 과 exploitation이 반복적으로 일어나며 이를 git과 같은 버전관리 방식으로 더 최적의 world condition을 찾아간다. Evolution에서 한 단계 더 나아간 것 같은 느낌
- 그래서 world에 적절한 agent 제약을 찾아간다는 점에서 명시적으로 공개되어있고 (정답을 알 수 있고) saturate되지 않았고 관심도가 높은 benchmark를 알아볼 필요가 있음
	- [[0_(world)_benchmarks]]
	- 

[^1]: 참고문헌 조사가 필요함. 필드에서 공감할 수 있는 용어를 사용해야함