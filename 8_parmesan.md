[ParmeSan: Sanitizer-guided Greybox Fuzzing](https://download.vusec.net/papers/parmesan_sec20.pdf)

# 0. Abstract
- fuzzing할때 취약점을 어디서 찾아야 하는지 중요함
- coverage-guide fuzzer는 bug coverage와 code c overage간의 상관관계를 고려하여 많은 code cover을 위하여 무차별적으로 최적화 함 하지만 *overapproximates*
- 이 접근방식은 이상적이지 않으며 TTE증가로 이어짐
- DF 잠재적 취약점이 있는 BB로 direction 하여 이 문재를 해결하여 TTE를 크게 줄일 수 있지만 bug coverage를 *underapproximate* 할 수 있음


- 이 논문에서 sanitizer-guided fuzzing을 제시함. bug coverage를 최적화 함
- 기존 saitizer에 의해 수행된 계측이 fuzzer-induced error dcondition에 사용되지만 더 나아가 흥미로운 BB를 감지하고 fuzzer을 가이드 할수 있다는 관찰을 함
- 이를 이용하여 sanitizer-guided fuzzer *ParmeSan*을 설계하구 구현하였다. 이는 TTE를 크게 줄이고 Coverage-base Fuzzer (*Angora*), DGF(*AFLGo*)보다 더 빠르게 동일한 bug를 찾는다.

# 1. Introduction
- code coverage based fuzzer : *overapproximates*
- directed fuzzer : *underapproximates*
  
- compile *sanitizer* : target program 의 bug 검사를 삽입하는 오류 검출 framework
- 기존에는 *sanitizer*를 사용하여 버그 감지와 분류를 함. 우리는 이를 이용하여 directed fuzzing을 수행함.

- 우리는 sanitizer-guided fuzzer *ParmeSan*을 만듬. 
- Sanitizer를 이용하여 target에 대한 bug coverage를 극대화함

challenges
1. sanitizer가 흥미로운 target을 자동으로 추출하는 방법
    
    버그를 포함할 가능성이 낮은 무의미한 검사를 제거하는 heuristic 사용
2.  fuzzing을 target으로 유도하기 위한 inter-procedural CFG를 구축하는 방법
    
    정적 구축은 부정확 하기 때문에 동적으로 구축함.

3. 구성요소 위에 fuzzer를 설계하는 방법
   
   *ParmeSan*은 target point에 대한 fuzzing, CFG 구성을 위한 fuzzing 두 단계로 이루어짐


이 논문의 기여

- 기존 compiler sanitizer pass에 의존하여 흥미로운 fuzzing target을 찾는 일반적인 방법
- input을 target으로 유도하기 위한 정확한 CFG를 동적으로 구축
- *ParmeSan* 구현
- *ParmeSan*을 평가하여 coverage guide fuzzer, DF와 비교하여 동일한 버그를 더 적은 시간에 찾는다는 것을 보여줌 

# 2. Background
## 2.1. Fuzzing strategy
- BF : 무작위 입력을 생성하여 crash를 유발함. 어떤 프로그램과도 쉽게 호환됨
- WF : SE와 같은 analysis를 사용하여 대량의 입력ㅇ이 아닌 버그를 유발하는 입력을 생성함. 확장성, 호환성 측면에 문제가 존재함.
- GF : BF의 확장 가능한 방식 이지만ㅇ input을 mutate 하기 위한 heuristic을 사용함

- *AFL* : coverage guided graybox fuzzer : 실행 추적 정보를 사용하여 input을 muation
- *Angora*, *VUzzer* : dynamic data-flow analysis (DFA)를 사용함

coverage guided fuzzing은 깊은 버그를 찾는데 오랜 시간이 걸릴 수 있음, 반면 DF 는 특정 target으로 유도

## 2.2. Directed fuzzing
https://chat.openai.com/c/8366002a-100a-44ed-a350-d3e10380e6ab
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