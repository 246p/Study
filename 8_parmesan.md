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

- DF는 잠재적으로 취약한 위치로 fuzzing 을 유도.
- 확장성과 호환성의 한계를 가진 SE를 이용함
- *AFLGo*의 DGF 개념은 GF의 확장성을 도입

두가지 문제
1. 흥미로운 타겟을 찾는것
2. 흥미로운 타겟 까지의 거리 계산


## 2.3. Target selection with sanitizers
- 현대의 compiler (*GCC*, *Clang+LLVM*)은 runtime check를 사용하여 정적 분석으로만 찾을 수 없는 가능한 버그를 탐지하는 *sanitizer*를 제공
- *sanitizer*는 fuzzer의 bug 찾기 능력을 향상시키는 데 사용됨, overhead가 큼
- BOF, UAF와 같은 취약점을 위한 검사를 추가함.
- ParmesSan은 버그 찾기 능력 뿐만아니라 TTE를 줄이기 위한 fuzzing 전략의 효율성 개선을 위해 사용됨

## 2.4. CFG construction
- DF는 seed를 선택할때 target까지의 거리를 고려함
- *AFLGo*, *Hawkeye*는 가벼운 정적 instrument를 사용하여 특정 seed와 target의 거리를 계산함
- *AFLGo*는 이로 인하여 CFG를 underapproximeating
- *Hawkeye*는 이를 해결하기 위하여 overapproximeating 하기 위하여 point-to analysis를 사용함
- context-sensitive, flow-sensitive는 무거움
- context-insensitive 는 가능하지 않은 실행 경로를 생성할 수 있음
- 이 문제를 해결하기위해 DFA로 보강된 동적 CFG구성을 제안
# 3. Overview
![figure1]()

## 3.1. Target acquisition
- fuzzer가 도달하기 희망하는 여러 target을 수집
- target set은 sanitizer의 instrumentation에 의해 생성됨
- 정적 분석을 통하여 basline과 instrumented version을 비교화여 sanitizer에 의해 배치된 instrumentations를 찾음
- 흥미로운 target을 찾기 위하여 pruning heuristic을 사용하여 더 작은 집합을 도출함
## 3.2. Dynamic CFG
- 동적 CFG는 "many-target directed fuzzing"에 적합한 input-aware CFG abstraction을 유지함. 
- 실행중 관촬 되는 대로 CFG에 edge를 추가하며 DFA를 사용하여 input과 CFG 사이의 의존성을 추적함
- 주어진 input에 대해 CFG에 영향을 줄 수 있는 input byte에 대한 feedback을 input mutation에 제공

## 3.3. Fuzzer
- *ParmeSan* fuzzer는 instrumented binary, target set, initial distance calculation, seed를 입력으로 받음
- input seed로 시작하여 실행된 BB의 initial set과 이 블록에 의해 커버된 조건을 얻음
- 동적 CFG에 의해 제공되는 정확한 거리정보를 이용하여 target들로 유도함
- 각 시도마다 target BB까지의 최적 거리를 결과로 하는 조건을 해결함

- CFG 구성에서 DFA를 사용하였기에 branch constraint를 해결하기위 DFA를 사용
- DFA-base coverage-guided fuzzer와 유사하게 sanitizer 검사를 뒤집고 bug를 trigger하는데 사용됨

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