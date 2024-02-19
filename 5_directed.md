[Directed Greybox Fuzzing](https://acmccs.github.io/papers/p2329-bohmeAemb.pdf)

# 0. Abstract
기존의 Graybox Fuzzer(GF)는 문제가 있는 변경이나 패치, 중요한 system call, 위험한 위치, stack trace에 있는 함수등 특정 방향으로 효율적으로 fuzzing할 수 없다.

이 논문에서는 주어진 프로그램의 위치에 효율적으로 도달하는 입력을 생성하는 Directed Graybox Fuzzing(DGF)를 소개한다.

taget 위치에 더 가까운 seed에 많은 에너지를 할당하고 멀리 있는 seed에 에너지를 줄이는 annealing-based power schedule을 개발하고 평가한다. 우리의 구현체인 *AFLGo*로 수행한 실험은 DGF가 symbolic execution 기반 whitebox fuzzing 및 undirected graybox fuzzing보다 우수하다는것을 보여준다. DGF를 patch testing, crash reproduction에 적용한 사례를 보여주고 *AFLGo*를 Google의 fuzzing platform *OSS-Fuzz*와 통합하는 방법을 논의한다. directed로 인하여 *AFLGo*는 *LibXML2*와 같이 잘 fuzzing 된 프로젝트에서 39개의 bug를 찾았고 17개의 CVE가 할당되었다.
# 1. Introduction
GF는 가벼운 instrumentation을 사용하여 아주 작은 성능 overhead로 input에 의해 실행되는 path에 대한 unique identifier을 결정한다.

새로운 input은 제공된 seed input을 mutation하여 새로운 path를 실행하는 경우 fuzzer의 queue에 추가된다. *AFL*은 수백개의 취약점을 발견하는데 기여하였고 허공에서 유효한 이미지 파일을 생성할 수 있다는 것이 입증되었고 이를 확장하는데 관여한 큰 커뮤니티를 가지고 있다.

기존 GF는 directed될 수 없다. undirected fuzzer와 달리 directed fuzzer는 대부분의 시간을 특정 taget 위치에 도달하는데 사용된다. directed fuzzer의 적용 사례는 다음과 같다.

1. patch testing 
   
   변경된 statment를 taget으로 설정하여 path testing을 수행하면 중요한 구성 요소가 변경될때 이것이 취약점을 도입하였는지 확인할 수 있다. 
2. crash reproduction
   
   stack trace의 method call을 taget으로 설정하여 crash reproduction을 수행할 수 있다. 현장에서 crash가 보고될때 stack trace와 일부 환경 매개변수만 내부 개발팀에 전송된다. 사용자의 개인정보 보호를 위하여 구체적인 crash input은 종종 사용될 수 없다. directed fuzzer는 내부 팀이 crash를 reproduction 할 수 있게 해준다.
3. static analysis report verification
   
   static analysis tool이 잠재적으로 위험하다고 보고하는 statement를 taget으로 설정하여 static analysis report verification을 수행한다. 
4. information flow detection

    민감한 source와 sink를 taget으로 설정하여 information flow detection을 수행한다. 데이터 유출 취약점을 노출하고자 하는 연구자는 개인정보를 포함하는 민감한 소스를 실행하고 데이터가 외부 세계에 보이는 민감한 sink를 실행하는 실행을 생성하려고 한다. directed fuzzer는 이러한 실행을 효율적으로 생성하는데 사용될 수 있다.


## 1.1. symbolic execution
기존 대부분의 directed fuzzer는 SE에 기반하고 있다. SE는 program analysis와 constraint solving을 사용하여 다른 path르 실행하는 입력을 생성하는 WF 기술이다. directed fuzzer를 구현하기 위하여 SE는 체계적인 path serach 때문에 선택되었다.

control-flow graph에서 taget 위치로의 path를 π라 할때 SE는 π를 실행하는 모든 입력에 만족하는 논리공식 φ(π)를 구성할 수 있다. SMT solver는 path contraint φ(π)를 만족하는 경우 path contraint에 대한 solution인 input t를 생성한다. 즉 t는 π를 실행하는 input 이다.

DSE는 도달 가능성 문제를 constraint satisfaction problem으로 다룬다. 대부분의 path가 실제로는 실행 불간읗 하기 때문에 검색은 일반적으로 중간 taget에 도달하는 실행 가능한 path를 반복적으로 찾는다. 

![figure1]()

예를들어 patch testing tool (Katch)는 SE engine (Klee)를 사용하여 변경된 문장에 도달한다. 위 그림의 1480 line을 taget으로 설정하고 Katch가 1465 line 까지 도달하는 경로 π0를 찾았다고 가정하자. Katch는 φ(π0)&(hbtype == TLS1_HB_REQUEST)를 constraint solver에 전달하여 실제로 1480 line을 실행하는 input을 생성한다. GF와 달리 SEBWF는 directed fuzzing을 구현하는 간단한 방법을 제공한다.

DSE는 program analysis와 constraint solving에 상당한 시간을 소비한다. 따라서 DSE가 하나의 input을 생성할때 GF는 수십배의 input을 생성할 수 있다. 

이 논문에서는 DGF를 소개한다. 도달 가능성 문제를 최적화 문제로 보고 생성된 seed가 taget까지 거리를 최소화 하기 위해 특정 meta-heuristic 을 사용하여 seed 거리를 계산하기 위해 각 basic block에서 taget까지 거리를 계측한다. seed distance는 procdural 하지만 우리의 새로운 측정은 call graph에서 한번, 각 CFG에서 한번만 분석gksek. runtime에서  fuzzer는 실행된 basic block의 거리의 평균으로 seed distance를 계산한다.

DGF가 seed 거리를 최소하기 위하여 mata-heuristic은 *simulated annealing*으로 불리면 power schedule로 구현된다. power schedule은 모든 seed의 energy를 제어하고 seed energy는 fuzzing하는데 사용된 시간을 지정한다. 다른 GF와 마찬가지로 analysis를 compile time으로 이동함으로서 runtime의 overhead를 최소화 한다.


## 1.2. directed graybox fuzzing
우리의 실험은 DGF가 DSE보다 효과성, 효율성 측면에서 우수함을 보여준다. 하지만 두 기술의 통합은 각 기술을 개별적으로 사용하는것보다 더 우수하다. 우리는 AFL에 DGF를 구현한 *AFLGo*를 만들었다. patch testing을 위하여 *AFLGo* 를 *Katch*와 비교하여 더 좋은 성능을 보였다.

crash reproduction을 위하여 *BugRedux*와 비교하여 3배 더 좋은 성능을 보였다.

우리의 실험은 simulated annealing 기반 power scheduling이 효과적이며 *AFLGo*가 효과적으로 directed 할 수 있음을 보여준다. AFL에 비해 AFLGo가 더 좋은 결과를 얻었다.

또한 AFLGo와 OSS-Fuzz를 통합하였는데 더 좋은 성능을 얻었다.
## 1.3. main contribution
- GF와 simulated annealing의 통합
- inter-rocedural distance의 측정
- *AFLGo*로 구현된 DGF
- *OSS-Fuzz*와 *AFLGo*의 통합
- DGF의 patch testing, crash reproduction 에서의 효과성에 대한 평가

# 2. Motivating Example
우리는 Heartbleed 취약점을 case study와 motivating sexample로 사용하여 DF에 대한 두가지 접근 방법을 논의한다. 전통적으로 DF는 SE에 기반한다. 우리는 SE engine *Klee*에 기반한 patch testing tool *Katch*와 이 논문에서 제시된 DGF 구현체 *AFLGo*를 비교한다.

## 2.1. Heartbleed and Patch Testing
Heartbleed (CVE-2014-0160)은 명빅히 안전한 protcol (SSL/TLS)을 통해 전송된 데이터의 개인정보 및 무결성을 위협하는 취약점이다. 이것은 man in the middle (MITM) 없이 공격 가능하다. 이 버그는 OpenSSL library에 도입된 2년이 지난 후에 탐지되었으며 광범위하게 분포가 발생하였다.

2011 새해 첫날 새로운 기능을 구현한 commit 4817504d에 의해 도입되엇다. 변경된 문장들을 taget 위치로 삼는 DF가 이를 발견하였다면 광범위한 확산을 막을 수 있었을 것이다. 현재 *OpenSS*L은 50만줄의 코드로 이루어져 있다. 해당 commit은 500줄 이상의 코드를 추가하였다. 실제로 최근 변경사항만이 오류를 발생하기 쉬운 것으로 간주되는 상황에서 OpenSSL 전체를 undirected fuzzing을 수행하는것은 어려울 것이다. DF는 이러한 변경사항을 훨씬 더 효율적으로 실행할 것이다. 대부분의 patch testing tool (*Katch*, *PRV*, *MATRIX*, *CIE*, *DiSE*, *Qietal.`s*)등은 SE를 기반으로 한다. 이중 *Katch*를 대표로 비교하였다.

## 2.2.  Fuzzing the Heartbleed-Introducing Source Code Commit
### 2.2.1. Katch
*Katch*는 *Klee* 위에서 구현된 patch testing tool이자 DSE engine 이다. 먼저 *OpenSSL*은 *LLVM 2.9*로 컴파일 되어야 한다. *Katch*는 한번에 변경된 basic block 하나를 처리한다. 우리의 regression test suite에 의해 cover되지 않은 도달 가능한 taget 위치로 11개의 변경된 basic block을 식별한다. 각 taget  `t`에 대해 *Katch*는 다음과 같은 *greedy search*를 수행한다. 
1. *Katch*는 regression test suite에서 `t`에 가장 가까운 seed `s`를 식별한다.
2. program analysis를 사용하여 `t`에서 가장 가까운 실행 분기 `b_i` 를 식별하고 `s`가 실행하는 모든 분기 조건의 연속인 path constraint `Π(s) = φ(b0)∧φ(b1)∧..∧φ(bi)∧...` 를 구성한다.
3. `b_i`를 부정하기 위해 수정해야 하는 `s`의 구체적인 input byte를 식별한다.
4. `b_i`의 조건을 부정하는 `Π'(s) = φ(b0)∧φ(b1)∧..∧¬φ(bi)`을 도출하고 이를 *Z3*(SMT solver)를 이용하여 concrete 한 input byte를 계산한다.

![figure2]()

### 2.2.2. Challenge

DSE 기반 WF는 효과적이지만 비용이 매우 많이 든다. 우리의 실험해서 *Katch*는 24시간 동안 heartbleed를 탐지하지 못하였다. 탐색의 모든 새로운 path에 대해 distance가 runtime에 다시 계산된다.interpreter가 모든 byte code를 지원하지 않고 path constraint가 floating point arithmetic과 같은 모든 언어 기능을 지원하지 않을 수 있기에 검색이 불완전할 수 있다. 왜냐하면 sequential search로 인하여 매 taget 마다 검색이 새로 시작된다.

### 2.2.3. Opportunities
*AFLGo*는 매우 효율적인 DGF이다. 초당 수천개의 입력을 생성하고 실행하여 Heartbleed를 20분 내에 발견한다. 또한 runtime에 program analysis가 사실상 필요 없고 compile/instrumentation time에 가벼운 program analysis를 수행한다. 또한 simulated annealing에 기반한 grobal search를 수행한다. 이를 통해 global optimum에 접근하여 taget 위치에 도달할 수 있다. 검색이 빠르게 global optimum에 도달하는것이 중요하다. 

*AFLGo*는 모든 taget을 동시에 검삭하는 parallel search를 이용한다. seed `s`가 더 많은 taget 을 실행할수록 `s`의 적합도를 더 높게 평가한다.

먼저 `AFLGo`는 *OpenSSL*을 계측한다. *Clang*에 대한 추가 compiler pass는 컴파일된 바이너리에 *classic AFL*및 *AFLGo*계측을 추가한다. *AFL* 계측은 code coverage 증가에 대해 알리고 *AFLGo*계측은 fuzzer에 주어진 taget 과 실행된 seed의 거리를 알린다. 새로운 거리 계산은 [3.2절](#32-a-measure-of-distance-between-a-seed-input-and-multiple-target-locations)에서 논의되며 *Katch*보다 훨씬 복잡하다. 왜냐하면 모든 taget 을 동시에 고려하여 compile time동안 설정되어 runtime의 overhead를 줄인다.

이후 *AFLGo*는 simulated annealing을 사용하여 *OpenSSL*을 fuzzing한다. 처음에 *AFLGo*는 exploration 단계에 들어가서 *AFL*과 동일하게 작동한다. exploration 단계에서 제공된 seed들 무작위로 mutation하여 많은 새로운 input을 생성한다. 만약 새로운 input이 code coverage를 증가시키면 seed set에 추가되고 그렇지 않으면 버려진다. 제공된 seed와 생성된 seed는 continuous loop에서 fuzzing된다. 예를 들어 Figure 2 에서 branch $(b_0,b')$을 실행하는 $s_0$과 $(b,b_1,b'')$을 실행하는 $s_1$인 $s_0, s_1$을 두 seed로 시작한다 하자. 시작할때 같은 수의 새 input이 각 seed에서 생성될것이다. 여기서 $s_0$이 t와 더 가깝지만 $s_1$의 자식들이 도달할 확률이 더 높을 것이다.

*AFLGo*가 exploitation 단계에 들어가는 시간은 사용자에 의해 정해진다. 우리는 실험을 위해 24시간의 timeout중 20시간을 설정하였다. *AFLGo*는 taget에 더 가까운 seed에서 더 많은 input을 생성한다. 너무 멀리 떨어진 seed를 fuzzing 하는 시간을 낭비하지 않는 것이다. 이 시점에서 branch $(b_0, b_1, b_i, b_{i+1}$ 을 실행하는 seed $s_2$를 생성했다고 해보자. explolation 단계에서 대부분은 시간은 t와 가까운 $s_2$를 fuzzing 하는데 사용된다. *AFLGo*는 simulated annealing function으로 구현된 power schedule에 따라 explolation 단계에서 exploitation 단계로 전환된다.

# 3. Technique
우리는 사용자가 정의한 위치에 중점을 둔 취약점 탐지 기술인 DGF를 개발한다. DGF는 모든 program analysis가 compile time에 수행되기 때문에 runtime중 어떤 프로그램 분석도 수행하지 않아 graybox fuzzing의 효율성을 유지한다. DGF는 필요할때 더 많은 computing power을 사용할 수 있도록 parallelizable하다. 

DGF는 여러 taget 위치를 지정할 수 있다. 우리는 instrumentation-time에 완전히 결정되어 runtime에 효율적으로 계산될 수 있는 *inter-procedural measure*을 정의하였다.

이이 측정은 실제 call graph(CG)와 control-flow-graphs(CFGs)를 기반으로 한 intra-procedural analysis이다. CG와 CFGs는 *LLVM compiler* 에서 쉽게 사용할 수 있다. 

이 새로운 거리 측정을 사용하여 가장 인기있는 annealing function인 exponential cooling schedule을 통합하는 새로운 power schedule을 정의한다. annealing-based power schedule은 거리 측정에 따라 taget 위치에 더 가까운 시드에 점차적으로 더 많은 에너지를 할당한다.

## 3.1. Greybox Fuzzing
오늘날 우리는 fuzzer를 program analysis 정도에 기반한 3가지로 구분한다.
- BF : 프로그램은 실행만 가능
- WF : SE기반으로 무거운 program analysis와 constraint solving을 요구
- GF : WF, BF사이에서 프로그램 구조의 일부를 얻기 위하여 경량 계측을 사용

*AFL*과 *LibFuzzer*와 같은 coverage-based GF(CGF)는 경량 계측을 사용하여 coverage 정보를 얻는다. 예를들어 *AFL*은 basic block transition과 함께 대략적인 분기 취득 횟수를 포착한다. *CGF*는 생성된 input중 어떤 것을 얼마나 오랫동안 fuzzing할지 결정할때 coverage 정보를 사용한다. 

우리는 이 계측을 확장하여 선택된 seed가 주어진 위치까지 거리를 고려하도록 한다. 거리 계산은 CG, CFGs에서 taget 노드까지의 최단 경로를 찾는것을 요구하면 이는 *LLVM*을 이용하여 *Dijkstra`s algorithm*을 사용한다.

Algorithm 1은 CGF가 작동하는 방식을 보여준다. 

![algorithm1]()

fuzzer는 seed input set `S`를 제공받아 timeout에 도달하거나 fuzzing이 중단될때까지 `S`에서 input `s`를 선택한다. (`chooseNext`) 예를 들어 *AFL*은 circular queue에서 seed를 선택한다. 선택된 seed input `s`에 의해 새로 생성할 입력의 수 `p`를 결정한다. (`assignEnergy`) 이 부분을 수정하여 annealing-based power schedule이 구현하였다.

그런 다음 fuzzer는 정의된 mutation operator에 의해 `s`를 무작위로 mutation 하여 `p`개의 새로운 input을 생성한다. (`mutate_input`) 생성된 input `s'`이 새로운 branch를 cover한다면 circular queue에 추가한다. 만약 crash를 유발한다면 crash input set `S_X`에 추가한다.

*Böhme*등은 CGF이 Markov chain으로 모델링 될 수 있음을 보여주었다. state i 는 프로그램의 특정 path이다. state i 에서 state j 로 전환하는 확률 $p_ij$는 path i를 실행하는 seed를 fuzzing하여 path j를 실행하는 seed를 생성할 확률이다.

저자는 CGF가 특정 path를 매우 자주 실행하는것을 발견하였다. stationary distribution 의 밀도는 일정 횟수의 반복 후 특정 path가 실행될 가능성을 의미한다.

## 3.2. A Measure of Distance between a Seed Input and Multiple Target Locations
함수간의 거리르 계산하기 위해 function level의 CG와 node와 basic block level의 CFGs에 값을 할당한다.

taget function $T_f$, taget block $T_b$은 주어진 소스 코드에서 신속하게 식별할 수 있다. function-level target distance는 CG내 두 function의 거리를 결정한다. 우리는 함수거리 $d_f(n,n')$ 를 CG내의 function $n, n'$사이의 최단 경로에 따른 edge의 수로 정의한다.


### 3.2.1. function
function n과 target function $T_f$사이의 `function-level target distance` $d_f(n,T_f)$를 도달 가능한 모든 target function $t_f ∈ T_f$의 조화평균으로 정의한다.

![expression1]()

$R(n,T_f)$가 CG에서 n에서 도달가능한 모든 target function의 집합일때 아래와 그램과 같은 이유로 조화평균을 사용한다.

![figure4]()


### 3.2.2. basic block
`basic-block-level target distance`는 basic block에서 함수를 호출하는 다른 모든 basic block 까지의 거리를 결정하며, 호출된 함수에 대해 function-level target distance의 곱을 고려한다.

직관적으로 target으로 향하는 call chain에서 함수를 call 하는 다른 basic block 까지의 평균 거리를 기반으로 한다. 또한 call chain이 짧다면 할당된 distance가 더 작다. 
우리는 BB거리 $d_b(m_1,m_2)$를 CFG $G_i$ 내의 basic block $m_1, m_2$ 사이의 최단 경로의 edge 수로 정의한다.

N(m)은 basic block m이 호출하는 함수의 집합으로 정의한다. 이때 $∀n ∈ N(m).R(n, T_f) \not = ∅$, $∀m ∈ T.N(m)  \not= ∅$ 이다. 따라서 다음과 같이 basic-block-level target distance를 정의한다.

![expression2]()

### 3.2.3. seed
시드 s 에서 target set $T_b$까지의 거리인 `normalized seed distance`




## 3.3. Annealing-based Power Schedules

## 3.4. Scalability of Directed Greybox Fuzzing

# 4. Evaluation Setup

## 4.1. Implementation

## 4.2. Infrastructure

# 5. Application 1 : Path Testing

## 5.1. Patch Coverage

## 5.2. Vulnerability Detection

# 6. Application 2 : Continuous Fuzzing

## 6.1. LibXML2

## 6.2. LibMing

# 7. Application 3 : Crash Reproduction

## 7.1. Is DGF Really Directed?

## 7.2. Does DGF Outperform the Symbolic Execution-based State of the Art?

# 8. Threats to Validity

# 9. Related Work

# 10. Conclusion
*AFL*과 같은 Coverage-based GF는 프로그램 analysis의 부담 없이 많은 path를 cover하려고 한다. *Klee*, *Katch*와 같은 SE 기반 WF는 필요할때 symblic program analysis와 constraint solving을 사용하여 검색을 정확하게 지시한다.

SE는 directed fuzzer를 구현하기 위한 기술로 선택되어 왔다. 주어진 대상에 도달하는 것은 path constraint를 해결하는 문제이다. SE는 특정 path를 탐색하기 위하여 수학적으로 엄밀한 framework를 제공한다. 반면 GF는 본질적으로 무작위적이며 box 밖에서 direction을 지원하지 않는다. 근본적으로 GF도 단지 random seed file 에서 random mutation을 적용할 뿐이다.

이 논문에서 이러한 direction을 GF에 도입하려고 시도하였다. runtime에서 GF의 효율성을 유지하기 위해 대부분의 프로그램 분석을 계측시간으로 옮기고 test generation중에 simulated annealing으로 global meta-heuristic을 사용한다. DGF는 DSE와 달리 무거운 프로그램을 분석할때 runtime overhead나 instruction을 path constraint로 인코딩 하지 않는다.

우리의 GF인 *AFLGo*는 단 몇천줄의 코드로 구현되었고 설정이 쉽다. SE와 달리 DGF는 무작위적이며 주어진 목표로 체계적으로 유도될 수 없다. 따라서 *AFLGo*가 기존의 DSE 기반 WF보다 성능이 좋은것은 놀라운 일이다. 효과성(더 많은 취약점), 효율성(같은 시간 더 많은 목표) 측면에서 모두 우수하다.

DGF는 다양한 방법으로 사용될 수 있다.
- 문제가 되는 변경이나 패치
- 중요한 system call
- 보고된 취약성의 stack trace
- crash 재현
  
미래의 작업으로 SEBBWF과 DGF를 통합할 것이다. SE는 최적화 문제로 serach를 모두 활용하는 통합된 DF은 그들이 결합된 강점을 활용하여 개별적인 약점을 완화할 수 있을 것이다. 실험에서 입증된것과 같이 잠재적으로 취약한 프로그램 변경을 밝혀내기 위한 더 효과적인 patch testing으로 이어진다.

우리는 위험한 위치나 보안에 중요한 구성 요소를 지적하는 static analysis tool과 통합될때 *AFLGo*의 효과성을 평가할 것이다.



# 11. 의문점
만약 commit된 부분으로 인하여 기존 코드에서 발생하는 취약점을 탐지하는 방법?