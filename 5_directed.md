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
seed s 에서 target set $T_b$까지의 거리인 `seed distance` 를 다음과 같이 정의한다.

![expression3]()

여기서 `ξ(s)`는 seed s 의 execution trace 이다. 이는 실행된 basic block을 포함한다. 우리는 `normalized seed distance` = $bar{d}(s,T_b)$를 s에서 $T_b$까지의 거리의 차이로 다음과 같이 정의한다. 

![expression3_6]()


$bar{d}$는 [0,1] 범위에 있다. (normalized) 또한 다음과 같이 거리를 계산할때 필요한 heavy-weight program analysis를 instrumentation-time으로 옮겨 runtime 에서의 overhead를 줄이다.

1. CG와 CGFs를 추출한다. (compiler, bit code translation)
2. target이 주어진다면 function-level, basic-block-level target distance는 instrumentation-time에 계산된다.
3. normalized seed distance는 runtime에 계산된다.

## 3.3. Annealing-based Power Schedules
우리는 APS(Annealing-based Power Schedules)를 개발하였다. 기존 연구에서 GF를 power schedule을 통하여 Markov cahin으로 볼 수 있음을 보여주었다. 우리는 MCMC(Markov Chain Monte Carlo) optimization technique (ex Simulated Annealing)을 사용하여 더 가까운 seed에 더 많은 에너지를 할당하며 이 에너지의 차이는 시간이 지남에 따라 증가한다.
### 3.3.1. Simulated Annealing
SA algorithm은 점진적으로 global optimal solution set에 수렴한다. 우리의 경우 이 set은 최대 수의 대상 위치를 실행하는 seed의 집합니다. SA는 큰 이산적인 검색 공간에서 global optimal solution을 찾는 MCMC이다. SA의 특징은 더 나은 solution을 수용할 수 있지만 때로 더 나쁜 해결책을 수용할 수 있다는 점이다.*temperature*는 SA alogorithm의 매개 변수로 더 나쁜 해결책의 수용을 cooling schedule에 따라 감소한다.

cooling schedule은 수렴률을 제어한다. 초기온도 $T_0=1$과 온도 사이클 k의 함수이다. temperature는 모든 seed의 속성으로 작용하며 다음은 가장 인기있는 exponential cooling schedule이다.

![expression7]()

### 3.3.2. Annealing-based power schedule
취약점 탐지에는 timeout이 존재하기에 충분한 `exploitation` 후 `exploration` 단계에 들어가는 시간 $t_x$를 지정하고 싶다. 직관적으로 이 시간은 고전적인 gradient descent algorithm을 사용할 수 있지만 우리는 simulated annealing을 사용하였다. 

우리는 다음과 같이 $T_k <0.05$일때 exploration 단계에 들어가도록 하였다.

![expression8_11]()

또한 expontential cooling schedule을 이용하여 APS를 정의하였다. seed s와 target $T_b$가 주어졌을때 APS는 energy p 를 다음과 같이 할당한다.

![expression12]()

다음 그림과 같이 검색을 시작할때 가까운 곳과 먼곳에 동일한 에너지를 할당한다. 시간이 진행됨에 따라 가까운 시드에 더 맣ㄴ은 에너지를 할당한다.

![figure5]()

### 3.3.3. Practical Integration
*AFL*은 이미 실행 시간, 입력 크기, 발견된 시점, 가진 조상의 수에 기반하여 에너지를 할당한다. AFL의 기존 power schdule을 우리의 APS와 통합하여 정의하려고 한다. $p_{afl}(s)$가 일반적으로 seed s에 할당하는 에너지라고 하자. 우리는 통합된 APS $\hat{p}(s,T_b)$를 다음과 같이 계산한다.

![expression13]()

`annealing-based power factor` $f=2^{10(p(s,T_b)-0.5)}$는 AFL의 PS에 의해 할당된 에너지의 증가 또는 감소를 조절한다. figure 6에서 annealing-based power factor의 동작을 NSD $\bar{d}(s,T_b)$의 두 극단적인 경우에 따라 확인할 수 있다.

![figure6]()


> $\bar{d}(s,T_b)=1$ : 먼 seed

t=0일때 $f=1$이므로 AFL과 동일한 에너지를 할당 받는다. ($\hat{p}(s,T_b)=p_{afl}$)

하지만 10분이 지나면 원래 에너지의 15%만 할당 받는다. 

![expression14]()

위와 같이 target이 도달하기에 매우 멀리 있는 seed는 점점 적은 에너지가 할당되어 원래 에너지 $p_{afl}$의 1/32만 할당받게 된다.

> $\bar{d}(s,T_b)=0$ : 가까운 seed

이때도 마찬가지로 t=0일때 최대 거리를 가진 시드와 동일한 에너지를 할당받는다.

![expression15]()

대상 위치와 가까운 시드는 $p_{afl}$의 32배까지 점점 더 많은 에너지를 할당받는다.


## 3.4. Scalability of Directed Greybox Fuzzing
GF의 이점은 program analysis가 없기에 효율성이 좋다는 것이다. 이로 인해 짧은 시간에 매우 많은 input을 생성하고 실행한다. DGF는 일부 program analyiss를 수행한다. DGF의 scale에 대해 생각해보자.

우리의 distance measure는 inter-procedural이지만 program analysis는 intra-procedural이다. 이는 절약을 제공한다. inter-procedural에 대한 경험은 다음과 같다.

초기의 경우 *AFLGo*는 하나의 함수의 모든 호출 위치를 basic block과 연결하여 iCGF(inter-procedural CGF)를 먼저 구성한다. (몇시간 걸림), iCGF가 완성되면 모든 basic block에서 target까지의 평균 최단 경로 길이로 target distance를 계산한다. (node가 많기에 몇시간 걸림)
  
지금의 *AFLGo*는 iCFG 계산을 생략하고 CG에서 function-level target distance를 계산하고 basic-block level target distance를 call site에 대해서만 계산한 후 intra-precedural CFG를 계산한다. call site에서 function-level target distance는 target까지의 나머지 path에 대한 근사치로 사용된다.

중요한 것은 BB수준 target distance는 Djikstra algorithm에 의존하여 $O(V^2)$ 이다. 평균 m 노드를 가진 n개의 iCFG의 경우 $O(n^2m^2)$이다. 반면에 CG와 iCFG의 경우 우리의 계산은 $O(n^2+nm^2)$이다.

또한 대부분의 무거운 분석을 compile time(instrumentation time)으로 옮길 수 있도록 DGF를 설계하였기 때문에 runtime에서의 효율성을 유지한다.

1. compile time

AFLGo는 각 함수의 모든 basic-block에 대한 basic-block-level target distance가 계산된다. AFL trampoline은 BB에 BBTD를 추가한다. trampoline은 instrumentation을 구현하는 일련의 명령어이다. instrumentation framwork (LLVM)은 대규모 프로그램에 대한 static analyiss를 효율적으로 처리할 수 있다.
   
2. run time
AFLGo는 AFL과 동일하게 확장 가능하다. (*Firefox, PHP, Acrobat Reader*) AFLGo trampoline은 BBLTD 값과 실행된 BB를 집계하여 기존 AFL trampoline 대비 몇개의 명령만 추가하였다. ASP는 fuzzer에 구현하였기 때문에 AFL과 동일하게 확장 가능한것은 당연하다.

요약하자면 우리는 program analysis를 instrumentation time으로 옮겨 runtime의 효율성을 유지한다. 이동안 intra-procedural measure를 통하여 경량화 하였다. 


※ intra = 함수 내 분석, inter = 함수 간 분석
# 4. Evaluation Setup
DGF의 효율성과 유용성을 평가하기 위하여 여러 실험을 수행하였다. DGF를 기존의 undirected GF인 *AFL*에 구현하여 *AFLGo*라 명명하고 두가지 다른 문제 영역인 path testing과 crash reproduction에 적용하고 *OSS-Fuzz*와 통합하였다.

## 4.1. Implementation
AFLGo는 4가지 구성요소로 구현된다. Graph Extracgtor, Distance Calculator, Instrumentor, Fuzzer

*OSS-Fuzz*와 통합하기 위하여 이러한 구성 요소들이 원래의 빌드 환경 (make or ninja)에서 원할하게 통합될 수 있음을 보여준다 전체 아키텍쳐는 다음과 같다.

![figure7]()


### 4.1.1. Graph Extracgtor (GE)
CG와 CFGs를 생성한다. CG node는 function signature로 구분되며 CFG는 해당 basic block에 해당하는 source code의 첫 statement 로 구분된다. GE는 compiler(*afl-clang-fast*)에 의해 확장되는 *AFL LLVM Pass*의 확장으로 구현되어 있다.

### 4.1.2. Distance Calculator (DC)
CG와 intra-procedural control-flow를 사용하여 BB의 inter-procedural distance를 계산한다. DC는 *networkx*를 사용하여 garph를 parsing하고 Djikstra algorithm에 따른 최단 거리 계산을 수행하는 python script로 구현된다. DC는 BB-distance file을 생성한다.
### 4.1.3. Instrumentor
BB-distance file을 가져와 대상 binary의 각 BB를 instrument한다. 구쳊거으로 각BB에 대해 BBLTD를 결정하고 확장 trampoline을 주입한다.

trampoline은 jump 명령 후에 실행되는 assembly code 조각으로 cover된 control flow edge를 추적하는데 사용된다. edge는 64kb shared memory로 식별된다.

64bit architecture에서 우리의 확장은 shared memory의 추가적인 16byte를 사용한다. (dsdistance를 누적하기위한 8 + BB의 수를 기록하기 위한 8)

각 BB에 대해서 AFLGo Instrumentor 다음 코드를 추가한다.
1. 현재 누적된 거리를 로드하고 BB의 target distance를 추가하는 코드
2. 수행된 BB 수를 로드하고 증가시키는 코드
3. 두 값을 shared memory에 저장하는 코드

이는 *AFL LLVM pass*의 확장으로 구현되며 compiler는 *afl-clang-fast*로 설정되며 BB-distance file을 참조하여 ASAN(AddressSanitizer)과 함께 빌드된다. 
### 4.1.4. Fuzzer
AFLGo는 AFL 2.40b(AFLFast의 explore schedule을 사용)에 구현되어 있으며 이는 우리의 APS에 따라 instrumente된 binary code를 fuzzing 한다.
## 4.2. Infrastructure
22코어를 사용, 다른 fuzzer와 동일한 seed corpus(없는 경우 empty file)을 사용하여 실험
# 5. Application 1 : Path Testing
path testing에서 DGF의 적용을 보여주고 최신 path testing tool (*Katch*)와 *AFLGo*를 비교한다. 
이 실험에서 우리는 patch coverage와 vulnerability detection 측면에서 비교하였다. *Katch*의 저자들과 동일한 대상, 실험 파라미터, 인프라를 사용한다. 하지만 임의의 명령을 실행하고 임의의 파일을 삭제할 수 있는 findutils는 제외하였다. 그러나 이러한 도구를 비교할때 실증적 평가는 두가지 개념의 구현을 비교하는 것이지 개념 자체를 비교하는 것이 아니다. 효율성을 개선하거나 검색 공간을 확장하는것은 개념과 무관한 "engineering effort"의 문제일 수 있다. 우리는 관찰된 현상을 설명하고 개념적 기원과 기술적 기원을 구분하기 위해 노력을 하였다. 
## 5.1. Patch Coverage
우리는 *Katch*와 *AFLGo*모두가 달성한 patch coverage를 분석한다. 이는 각각 patch에서 변경된 이전에 cover 되지 않은 BB의 수로 측정된다. patch coverage는 새로운 또는 수정된 코드 영역이 테스트 과정에서 얼마나 자주 실행되는지를 나타내며 이는 새로운 취약점이나 문제점을 발견할 가능성을 높일 수 있다. 높은 patch coverage는 fuzzer가 효과적으로 patch된 code를 탐색하고 있음을 의미한다. *AFLGo*는 *Katch*보다 13% 많은 BB를 cover한다. 

cover되지 않은 target에 대해 분석한 결과 변경된 BB 절반 이상이 레지스터 간접 호출이나 점프 (function pointer)를 통해서만 접근 할 수 있었다. 이러한 호출이나 점프는 CG나 CFGs에서 edge로 나타나지 않는다. 또한 SE와 GF 모두 프로그램의 복잡한 입력 구조를 요구할때 큰 검색공간으로 인한 어려움을 겪는다. 

이러한 결과는 특정 기술의 한계를 극복하고 patch coverage를 개선하기 위한 추가적인 연구 개발의 필요성을 의미한다. 더 나은 품질의 regression test suite의 개발과 model-based fuzzing의 적용은 향후 연구의 방향성을 제공할 수 있다.

연구자들이 두가지 접근 방식에서 어떤 이익을 얻을 수 있는지 이해하기위해 두 기술 모두에서 cover된 target을 조사하였다. 아래 그림에서 확인할 수 있듯이 상대적으로 작은 교차점은 각 기술의 개별 강점 때문이라고 볼 수 있다.

![figure9]()


SE는 다른방식으로 접근하기 어려운 구역에 들어가기 위한 복잡한 constraint를 해결할 수 있다. 반면 GF는 특정 "이웃"의 경로에 갇히지 않고 target을 향해 많은 경로를 빠르게 탐색할 수 있다. *AFLGo*와 *Katch*는 서로를 보완한다.

이 결과는 보안 연구자들이 특정 취약점이나 코드 변경을 테스트할때 여러 기술을 조합하여 사용하는것의 중요성을 강조한다. SE와 GF의 결합은 보다 광범위한 coverage와 함께 다양한 유형의 취약점을 발견할 가능성을 높여준다. 이러한 포괄적인 보안 검증 프로세스를 이용하여 소프트웨어의 안정성을 향상시킬 수 있다.

미래에는 SE 기반 DWF와 DGF를 통합하여 사전에 지정된 target에 효과적이고 효율적으로 도달하는 DF를 달성할 계호기이다. 우리는 이러한 통합이 각 기술을 개별적으로 사용하는것보다 우월할것이라고 생각한다. 
## 5.2. Vulnerability Detection
*AFLGo*는 *Katch*에 의해 발견되 7개의 버그중 4개를 포함하여 이전에 보고되지 않은 13개의 버그를 추가로 찾아내었다. 이중 7개의 버그는 CVE-ID가 할당되었다.

우리의 DGF는 vulnerability detection 측면에서 최신 기술을 능가한다. 이 성공은 DGF 기법이 보안 검증 프로세스에서 중요한 역할을 할 수 있음을 보여준다.

우리는 이러한 버그의 발견이 target과 어떠한 관련이 있는지 조사하였다. 12개의 버그는 *AFLGo*의 ditextion의 결과로 발견되었다. 구체적인 7개의 버그는 target 위치를 포함한 stack trace를 가지고 있으며 다른 5개의 버그는 target 근처에서 발견되었다. 

*AFLGo*가 더 우수한 이유는 DGF의 효율성 때문이다. runtime에서의 program analysis가 필요하지 않으므로 더 많은 입력을 생성한다. 또한 *runtime checking*으로 인해 실행 중 crash를 발견하면 프로그램을 crash 한다. *Katch*는 *constraint-based error detction*을 이용한다.

우리는 target을 *runtime checker* (*ASAN*)으로 instrument 하였다. input이 허용되지 않은 메모리에 대한 읽기, 쓰기를 일으키는 경우, ASAN은 프로그램이 일반적으로 crash하지 않은 경우에도 SEGFAULT를 발생시킨다. fuzzer는 runtime checkr로부터 이 신호를 사용하여 생성된 input에 대한 오류를 보고한다.

# 6. Application 2 : Continuous Fuzzing
우리는 DGF의 유용성을 연구하기 위하여 *Google*의 *OSS-Fuzz*와 통합하였다. *OSS-Fuzz*는 continuous testing platform이다. 완전 자동화된 방식으로 등록된 프로젝트를 checkout, vuild, fuzzing 한다. bug report는 project 관리자에게 자동으로 제출된다. 

*AFLGo*를 통합하여 7개의 open-source project에서 26개의 구별되는 버그를 발견하였다. 즉 *AFLGo*는 well-fuzzed project 에서 취약점을 발견할 수 있는 *OSS-Fuzz*를 위한 patch testing tool 이다.

우리는 *LibXML2*와 *LibMing*에서의 발견된 버그에 대해 초점을 맞추어 설명한다.
## 6.1. LibXML2
*LibXML2*는 C언어로 작정된 널리 사용되는 XML parser library로 PHP의 핵심 요소이다. *LibXML2*의 최근 50개 commit을 fuzzing하여 4개의 구별되는 BOF를 발견하였다. 

우리는 두개의 crash를 incomplete bug fix로 식별하였다. bug report에 있는 input에 대해서는 수정되었지만 다른 input은 여전히 버그를 발생시킨다. 이 버그들을 수정하며 변경한 문장에 대해 target을 설정하여 *AFLGo*는 다른 crash input을 생성하였다.

다른 crash들 또한 동일한 commit의 인근에 존재한다.

> *AFLGo*는 수정되었다고 보도괸 곳에서 취약점을 발견하였다. 자주 패치되는 오류가 발생하기 쉬운 소프트웨어 요소에서 새로운 취약점을 발견할 수 있다.

## 6.2. LibMing
*LibMing*은 .swf 파일을 읽고 생성하기 위해 널리 사용되는 라이브러리로 *PHP v5.3.0*까지 PHP와 함꼐 번들로 제공되었다. 이것 역시 가장 최근의 50개 commit을 fuzzing함으로 *AFLGo*는 불완전한 수정을 발견하였다. 이 버그는 최근 다른 보안 연구자에 의해 발견되었고 CVE-ID를 받았다. 하지만 *AFLGo*는 정확히 동일한 stack trace를 생성할 수 있는 다른 crash input을 생성할 수 있었다. 즉 patch가 불완전한 것이다. 우리는 자세한 분석과 patch를 포함한 bug report를 제출하여 CVE-ID가 할당되었다. 

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