[ParmeSan: Sanitizer-guided Greybox Fuzzing](https://download.vusec.net/papers/parmesan_sec20.pdf)

# 0. Abstract
- fuzzing할때 취약점을 어디서 찾아야 하는지 중요함
- coverage-guide fuzzer는 bug coverage와 code c overage간의 상관관계를 고려하여 많은 code cover을 위하여 무차별적으로 최적화 함
- 이 접근방식은 이상적이지 않으며 TTE증가로 이어짐
- DF 잠재적 취약점이 있는 BB로 direction 하여 이 문재를 해결하여 TTE를 크게 줄일 수 있지만 bug coverage를 과소평가할 수 있음


- 이 논문에서 sanitizer-guided fuzzing을 제시함. bug coverage를 최적화 함
- 기존 saitizer에 의해 수행된 계측이 fuzzer-induced error dcondition에 사용되지만 더 나아가 흥미로운 BB를 감지하고 fuzzer을 가이드 할수 있다는 관찰을 함
- 이를 이용하여 sanitizer-guided fuzzer *ParmeSan*을 설계하구 구현하였다. 이는 TTE를 크게 줄이고 Coverage-base Fuzzer (*Angora*), DGF(*AFLGo*)보다 더 빠르게 동일한 bug를 찾는다.
# 1. Introduction
# 2. Background
## 2.1. Fuzzing strategy
## 2.2. Directed fuzzing
## 2.3. Target selection with sanitizers
## 2.4. CFG construction
# 3. Overview
## 3.1. Target acquisition
## 3.2. Dynamic CFG
## 3.3. Fuzzer
# 4. Target acquisition
## 4.1. Finding instrumented points

## 4.2. Sanitizer effectiveness
## 4.3. Profile-guided pruning
## 4.4. Complexity-based pruning

# 5. Dynamic CFG
## 5.1. CFG construction
## 5.2. Distance metric
## 5.3. Augmenting CFG with DFA
# 6. Sanitizer-guided fuzzer
## 6.1. DFA for fuzzing
## 6.2. Input prioritization
## 6.3. Efficient bug detection
## 6.4. End-to-end workflow
# 7. Implementation
## 7.1. Limitations
# 8. Evaluation
## 8.1. ParmeSan vs. directed fuzzers
## 8.2. Coverage-guided fuzzers
## 8.2. Sanitizer impact
## 8.4. New bugs
# 9. Related Work
# 10. Conclusion