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
*target acquisition*은 흥미로운 target을 찾기 위하여 compiler sanitizer에 의존한다.

- sanitizer의 error condition을 trigger 하여 direction을 함
- 우리 방식은 sanitizer-agnostic(중립) 하기에 sanitizer에 따라 다른 class의 bug를 대상으로 pipeline을 retarget할 수 있다.


## 4.1. Finding instrumented points
- Compiler framwork(ex *LLVM*)은 machine-agnostic intermediate representation (IR)로 변환함. 
- sanitizer 는 IR-level 에서 동작, IR을 계측하여 sanitization check를 추가


sanitizer는 두가지 방식으로 계측함
1. 내부 데이터 구조 업데이트 (shadow memory)
2. 실제 버그를 감지

- 우리는 sanitizer가 내부 데이터 구조를 업데이트할때 sanitizer에 의해 도입된 조건문 branch로 fuzzing을 유도 
- 이 branch로 들어간다는 것은 bug를 발견했다는것을 의미
- 다양한 anitizer가 존재하여서 우리는 sanitizer-agnostic한 방법을 원함 이를 위해 snitizer가 적용된 상태와 적용되지 않은 상태의 IR 차이에 대한 blackbox analysis를 구현하여 이를 수행
- 조건을 포함 하지 않은 BB를 포함시키기 위해 sanitizer에 의해 계측된 모든 BB를 추가함
- 조건을 포함하는 BB는 계측된 BB와 조건이 취해진 BB (sanitizer의 bug check 기능)을 포함시킨다.
- 이러한 방법으로 기존과 앞으로의 *LLVM sanitizer*와 호환되는 일반적이고 효과적인 target acquistion 가능
## 4.2. Sanitizer effectiveness
sanitizer instrmentation을 target으로 사용하는것을 확인하기 위하여 실제 버그를 탐지하는지 확인하였다.

- *AdreesSanitizer(ASan)* : BOF, UAF
- *UndefinedBehaviorSanitizer(UBSan)* : using nullptr, integer overflow
- *TypeSanitizer(TySan)* : type confusion (C/C++)


![table1]()

- 표는 sanitizer가 bug를 포착하는지와 계측된 BB으로 path가 포함되지 않은 BB의 수를 보여줌
- target으로 향하지 않는 nontarget BB의 수를 계산함
- *UBSan*, *TySan*은 많은 경우 code coverage의 상당부분을 무시할 수 있음
- *ASan*은 많은 BB를 계측하기 때문에 사실상 coverage-based fuzzing

challenge : target의 수를 제한 하면서도 intergesting target은 유지하는것 > 두가지 pruning heuristic을 도입
 
## 4.3. Profile-guided pruning
- target의 수를 제한하기 위하여 profile-guided pruning를 수행
- *ASAP*와 유사한 접근
- 프로그램을 프로파일링 하고 hot pass(profiling input으로 도달한 path)에 있는 모든 sanitizer를 제거함
- hot pass는 버그를 포함할 가능성이 낮다.
- 이 전략은 효과적으로 target set을 pruning 할 수 있다. 물론 일부 유효한 target을 제거할 수 있지만 *ASAP*의 저자는 보수적으로 80%의 버그가 감지된다고 주장한다.
## 4.4. Complexity-based pruning
sanitizer가 단순한 branch 외에도 다른 instrumentation을 자주 추가하기 때문에 추가/수정된 명령어 수를 기반으로 함수를 점수화 하고 더 높은 점수를 받은 target을 더 흥미로운것으로 표시한다.

직관은 sanitizer에 의해 변경된 명령어가 많을 수록 함수의 복잡도가 높고 sanitizer가 targeting 하는 bug class를 만날 가능성이 높다는 것이다.

*LAVA-M* 에서 *ASAN*을 사용하였을때 *base64*에서 상위 3개 target은 `lava_get(), lava_set(), emit_bug_reporting_address()`에 있다. 이중 상위 2개는 *LAVA-M*에서 bug를 발생시키는 함수이다.

점수는 profiling을 기반으로 puruning을 수행할때 고려된다.

※ cold code?

# 5. Dynamic CFG
*ParmaSan*이 [Target acquisition](#4-target-acquisition) 단계에서 식별한 code로 유도하기 위하여 정확한 CFG를 이용하여 BB와 target 사이의 distance를 추정할 수 있어야 한다.

- [CFG의 정밀도를 동적으로 향상시키는 방법](#51-cfg-construction)
- [execution trace와 target 사이의 거리를 구하는 방법](#52-distance-metric)
- [DFA를 추가하여 distance matric을 향상하는방법](#53-augmenting-cfg-with-dfa)
## 5.1. CFG construction
- 기존 DF는 정적으로 CFG를 구성하여 부정확한 결과
- 우리는 *LLVM*으로 구성된 *CFG*로 시작하여 fuzzing중 edge를 추가하여 점점 정밀하게 만듬 (런타임중 간접 호출은 정적으로 구성할 수 없음)
- 거리 계산을 하기 위하여 시작점과 target 사이의 조건문의 수를 사용함
- 전체 CFG에 비해 compact Condition Graph를 사용함
- 런타임에서 CG, CFG를 모두 사용하지만 거리 계산에는 Condition Graph만 사용함
- CFG의 node는 불변, edge는 동적, 새로운 edge를 만날때마다 CFG, Condition Graph에 추가
## 5.2. Distance metric
거리 측정은 fuzzer가 target에 더 가까워지기 위해 CFG의 어떤 부분을 다음에 탐색해야할찌 결정

거리 계산은 scalability issues가 있기에 간단한 측정법을 사용함

- branch condition `c`와 흥미로운 BB로 이어지는 branch condition 까지의 거리를 `d(c)`로 정의
- target branch의 이웃 BB이 가중치 1을 갖게 하는 재귀적 접근을 사용 (*AFLGo*와 유사한 조화평균을 사용하여 계산)

![expreession1]()

- N(c)는 c에서 target으로 가는 경로가 있는 고려되지 않은 successor set

주어진 input에 대한 실행 trace가 주어졌을때 거리 측정법을 이용하여 어떤 branch를 flip할지 결정하여 실행을 흥미로운 BB로 유도한다. 이 방법은 단순하지만 잘 작동한다. 물론 더 나은 scheduling이 존재할 수 있다.
## 5.3. Augmenting CFG with DFA
동적 CFG는 input에 따라 간접 호출을 단일 대상으로 고정하여 거리 계산으 ㄹ개선할 수 있다.

간접 호출 대상을 결정하는 input byte를 알고 있따면 가접 호출 대상을 알 수 있도록 input byte를 고정할 수 있다.

이러한 계산은 우리의 거리 계산의 정밀도에 큰 영향을 미치며 많은 간접 호출이 있는 경우 유익하다.
# 6. Sanitizer-guided fuzzer
## 6.1. DFA for fuzzing
- 기존의 DGF는 simple distance metrics를 기반으로 input을 direction
- DFA 기반 coverage-guided fuzzer는 DFA를 주가하여 mutated input이 branch constraint를 쉽게 해결
- DFA는 새로 발견된 분기에 영향을 미치는 input byte offset을 추적하여 해당 offset을 mutation 한다
- 우리는 CFG를 구성할때 DFA를 사용하고 있기에 input mutation에도 DFA를 적용
- sanitizer check를 flip하기 위한 특별한 mutation 전략이 필요하지 않음
## 6.2. Input prioritization
- main fuzzing loop는 조건문과 해당 조건문을 발견한 seed로 구성된 항목을 포함하는 priority queue에서 항목을 꺼냄
- queue는 (runs, distancs)로 구성된 tuple에 기반하여 정렬
- 가장 낮은 priority를 꺼내어 DFA에 의해 제공된대로 조건문에 영향을 미치는 input byte에 우선순위를 두어 mutation 수행
- 입력에 대해 DFA로 계측된 실행을 수행하여 BB에 대한 taint 정보를 수집 (새로운 code coverage를 찾을때만 수집)
- 원본 seed가 한 라운드에서 여러번(30) mutation 후 CFG가 변경되었다면 업데이트된 distance와 함께 다시 큐에 넣음
## 6.3. Efficient bug detection
- *ParmaSan*은 analysis 목적으로 sanitizer를 사용함
- 물론 sanitizer 없이 fuzzing을 수행할 수 있음
- 간단하지만 효율적인 최적화 *lazysan*을 지원
- target에 도달하였을때 saintizer instrumentation의 요구에 따라 layzysan을 활성할 수 있고 그 외에는 uninstrumentation 을 실행할 수 있다.

## 6.4. End-to-end workflow
*ParmaSan*의 end-to-end fuzzing workflow는 3간계로 구성된다.
1. short coverage-oriented exploration tracing phase : trace를 수집하고 가능한 정확한 CFG를 구축한다.
2. directed exploration : 원하는 target에 도달하기 위한 condition을 해결하려고 함
3. exploitation phase : 지정된 target중 하나에 도달하면 시작되어 DFA-driven mutation 수행

2,3 phase는 서로 교촤되어 실행됨


# 7. Implementation
- *ParmeSan*은 coverage-guided fuzzer *Angora*위에 구성된다.
- Angora : Rust
- blackbox sanitizer analysis : python, LLVM pass
- *ParmeSan*의 pipline에 *AFLGo*를 통합하여 *AFLGo*를 사용할 수 있음
- [target acquisition](#4-target-acquisition)을 구현하기 위하여 *llvm-diff* 사용
- [profile guided pruning](#43-profile-guided-pruning)을 구현하기 위하여 *ASAP*위에 구현, 이를 확장하여 [complexity based pruning](#44-complexity-based-pruning)을 구현함
- 동적 CFG는 *Angora*를 기반으로함. 
- *Angora*를 수정하여 queue에서 target acquisition 단계에서 생성된 target까지의 거리에 따라 정렬함.
- 동적 CFG 요소를 추가하여 CFG constraint 수집을 가능하게 하고 coverage와 condition에 기반하여 target 까지의 거리를 계산함
- *LLVM compiler framwork*의 DataFlowSanitizer *DFSan*을 사용함

## 7.1. Limitations
- *ParmaSan*은 *LLVM IR*에 의존, 이론상 IR 없이 binary에 적용 가능
- 현재 analysis는 compiler sanitizer pass에 의존, 원시 binary의 경우 compiler sanitizer대신 binary hardening을 사용할 수 있음
- linking time에 수정사항을 추가하는 일부 sanitizer에 대한 문제를 발견

- *ParmeSan*이 발견하는 bug class는 사용된 sanitizer에 의존함. *ASan*같은 일부 sanitizer는 다양한 *bug class*탐지 가능.

# 8. Evaluation
- *ParmaSan*을 다른 DGF, coverage-guided fuzzer와 비교
- 동적 CFG 구축이 간접호출에 대한 문제를 어떻게 개선하는지 보여줌
- target acquistion은 compile과정의 일부이기 때문에 실행 시간에 포함 X

## 8.1. ParmeSan vs. directed fuzzers
- DFA의 사용이 DF를 어떻게 향상시키는지 보여줌
- *AFLGo, HawkEye*의 benchmark를 재현

![table2]()

- target을 수동으로 정하여 test
- TTE, 버그를 현저히 개선함
## 8.2. Coverage-guided fuzzers
- 우리의 전략이 최신의 coverage-guided fuzzer보다 더 많은 버그를 더 빠르게 찾는다. 
- 더 적은 TTE, coverage로 버그 찾기 가능하다.

![table3]()

![table4]()

## 8.2. Sanitizer impact
특정 sanitizer가 fuzzing pipeline의 결과에 어떤 영향을 미치는지 알아본다. 즉 특정 유형의 bug에 대한 fuzzing을 집중시킬 수 있다.

- Parmasan은 memory-leak bug에 취약하다. 이는 사용오딘 sanitizer analysis가 중요한 영향을 미친다는 의미
- target acquisition에 *ASan*을 사용한다면 invalid memory use에 집중하게 됨
- memory-leak bug에 집중하려면 LeakSanitizer *LSan*을 사용 

> *LSan*은 IR을 수정하지 않고 malloc과 같은 함수로의 library call을 가로챔, 우리는 dummy call을 삽입하는 LLVM pass를 생성하여 *LSan*과 동일한 동작을 유지하며 IR을 변경함

![table5]()

위 표는 target acquisition을 위해 사용된 sanitizer간 차이를 보여줌

- bug를 cover하고 최소한의 target set을 계측하는 sanitizer을 사용하는것이 더 빠르게 버그를 찾을 수 있음
- 특정 bug class에 대한 fuzzing을 집중하고자 할때 적절한 sanitizer를 선택하는 것이 매우 중요함
- 단순히 탐지할 수 있는 bug class 뿐만 아니라 fuzzing 과정으 ㅣ효율성에도 영향을 미침
## 8.4. New bugs
![table6]()
# 9. Related Work
- AFLGo와 달리 ParmeSan은 target acquisition analysis를 수행함
- *Hawkeye*는 static alias analysis를 사용하여 간접 호출을 도와주려고 함 
# 10. Conclusion
- sanitizer-guided GF *ParmeSan*을 제시
- sanitizer check로 fuzzing을 guide
- 기존에 존재하는 sanitizer을 사용
- taint-enhanced input mutation
- dynamic CFG construction