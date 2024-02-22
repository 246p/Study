[Hawkeye: Towards a Desired Directed Grey-box Fuzzer](https://chenbihuan.github.io/paper/ccs18-chen-hawkeye.pdf)

# 0. Abstract
- GF는 실제 프로그램을 테스트하는데 효과적인 접근방식
- 기존 GF는 사용자가 지정한 target을 향해 실행하는 기능이 부족
- DGF의 4가지 속성을 특징으로 하는 *Hwakeye* 제안
- target에 대한 새로운 정적 분석을 통해 CG, function, BB-level distance 정확하게 수집
- 정적 정보와 실행 트레이스를 바탕으로 seed를 평가하여 seed 우선순위, power schedule, mutation에 도움을 줌
- 성능이 좋았고 15개의 CVE ID 할당
# 1. Intoroduction
- PUT (program under testing) 내부 구조의 인식 여부에 따라 BF, WF, GF로 분류 최근 GF가 효과적
- 일반적인 GF는 제한시간 내에 많은 program state를 cover 해야함
- 하지만 특적 program state만 중요할때가 있음
- 즉 fuzzer가 PUT내의 특정 target site에 도달하도록 지형해야할 필요성이 있음
- *AFLGo*는 DGF로 target site로 도달 가능성을 최적화 문제로 보고 meta-heuristic(simulated annealing)을 사용하여 더 짧은 거리의 seed를 많이 실행함

바람직한 DGF의 4가지 속성과 해결

P1. 모든 trace를 고려하고 특정 trace애 편향되지 않는 거리 기반 매커니즘을 가지고 있어야 한다. 
- [정적 분석 결과를 이용하여 adjacent-function distance 강화](#42-adjacent-function-distance-augmentation)
- [funtion,BB-level 거리를 adjacent-function distance를 기반으로 계산](#43-directedness-utility-computation)
- [정적 분석 결과와 런타임 정보를 이용하여 target function의 BB trace distance, coverd function similarity를 계산](#44-power-scheduling)

P2. 정적 분석에서 overhead와 유용성 사이의 균형을 맞추어야한다.
- [CG, CFG에 기반한 분석 (function level reachability analysis), function pointer를 위한 point-to analysis(간접호출)](#41-graph-construction)

P3. target site에 신속하게 도달할 수 있도록 seed를 scheduling해야 한다.
- [BB trace distance와coverd function similarity를 사용한 pwer scheduling](#44-power-scheduling)
- [seed 우선순위 문제](#46-seed-prioritization)

P4. seed가 다른 program state를 cover 할때 적응형 변이 전략을 채택해야 한다.
- [reachability analysis, coverd funtion similarity에 기반한 mutation 전략](#45-adaptive-mutation)

*Hwakeye*는 target site에 도달하는 시간과 crash를 발견하는 시간 측면에서 최신 GF보다 우수함

*Oniguruma*, *MJS*등 프로젝트에서 41개의 crash를 찾고 15개의 CVE ID가 할당됨

이 논문의 주요 기여는 다음과 같음
1. DGF의 문제를 분석하고 4가지 바람직한 속성을 요약
2. power function을 측정함
3. power scheduling, adaptive mutation, seed prioritization을 사용하여 target site로 수렴 속도를 증가시키는 새로운 접근방법 제안
4. 이러한 아이디어를 새로운 fuzzing framework에 구현하여 crash reproduction, target site covering 에서 평가함
# 2. Desired Properties of DGF
DGF의 어려움에 대한 예제를 바탕으로 바람직한 속성을 제안하며 *AFLGo*를 검토함
## 2.1. Motivating Example
![figure1]()
![figure2]()

`CVE-2017-15023`의 불완전한 수정으로 인한  NULL pointer dereference 버그인 `CVE-2017-15939`와 관련된 두 실행 trace이다.

DGF의 경우 T로 이어지는 trace를 선호하지만 모든 trace를 놓칠 수 있다. 예를들어 *AFLGo*는 짧은 trace(a,e,T,Z)를 선호하여 문제있는 trace인 (a,b,c,d,T,Z)를 놓칠 수 있다. 실제로 *AFLGo*를 24시간씩 10번 실행하였지만 crash를 발견하지 못하였다.

기존의 DGF의 도전과제는 다음과 같다.
1. target function이 PUT의 여러곳에 나타날 수 있으며 다양한 trace가 있을 수 있다.
2. CG가 trace distance 계산에 영향을 미치므로 정확하게 구출해야 한다. 특히 함수간 간접 호출을 무시하면 안된다.


## 2.2. Desired Properties of Directed Fuzzing

P1. 특정 trace에 대한 편향을 피하고 모든 target trace를 고려하는 강력한 distance-based mechanism을 정의해야 함

- 정적 정보가 제공되지 않는 경우 fuzzer는 target이 실행되기 전 target trace에 대해 알지 못함, 이미 coveer되더라도 fuzzer는 다른 trace가 있는지 여부를 알 수 없음, 따라서 guiding mechanism은 모든 traget trace를 찾는데 도움을 주어야함
- 그러나 모든 target trace에 대한 인식만 있어선 안됨. 대상에 도달할 수 있는  trace가 다른 trace에 비해 많은 에너지를 할당받도록 적절하게 계산되어야 함.
  
P2. 정적분석에서 overhead와 유영성의 균형을 맞추어야함.

정적 분석의 이점
1. C/C++ 에서의 간접 호출 (function pointer, funtion object)의 경우 call site를 source-code나 binary instruction 에서 확인할 수 없음.
2. 모든 call을 동등하게 처리해선 안됨. 어떤 함수는 calling function에서 여러번 호출되며 이는 런타임에 호출될 가능성이 더 높음. 정적 분적 관점에서 이러한 상황을 구별할 방법을 제공해야함.

figure 2 에서 DGF의 바람직한 설계는
1) `a`가 간접적인 방법으로 `T`를 호출하는 경우 정적 분석은 간접호출을 포착해야함.
2) caller가 더 많은 branch에 나타나고 caller 내에서 더 자주 발생하는 경우 더 작은 거리를 주어야 함, 

P3. seed를 선택하고 scheduling 해야 한다.   
DGF의 목표는 빠르게 최대 coverage에 도달하는것이 아닌 target에 빠르게 도달하는 것이다.

이를 위해 power scheduling을 사용하여 더 가까운 seed에 더 많은 fuzzing energy를 할당해야 한다.

P4. 적응형 mutation 전략을 사용해야함
- 일반적인 GF는 bitwise flip, byte rewrite, chunk replacement와 같은 mutation 을 적용
- 세밀한 변형(bitwise flip), 거친 변형(chunk replacement)으로 분류될 수 있음

거친 변형이 trace를 크게 변경할 가능성이 높음. 따라서 이미 target site에 도달한 경우 거친 변형을 적게 가해야함

## 2.3. AFLGo’s Solution
### 2.3.1. P1
*AFLGo*에서 정의된 거리 공식에 따라 거리를 계산해 본다면 할당되는 에너지는 `(a,e,T,Z) > (a,e,f,Z) > (a,b,c,d,T,Z)` 이다. crash trace가 가장 적은 에너지를 받게 된다.
### 2.3.2. P2
*AFLGo*는 explicit CG 정보만 고려한다. 결과적으로 함수 포인터는 거리 계산에서 무시된다. 만약 target function이 함수 포인로 인해 호출된다면 실질적으로 direction이 되지 않는다.
또한  caller에 있는 동일한 callee를 한번만 계산하여 다양한 호출 패턴을 고려하지 않는다. 이때 function-level distance는 *Dikstra*를 사용하여 인접한 함수의 가중치를 1이라고 가정하여 거리 계산이 왜곡된다.
### 2.3.3. P3
*AFLGo*는 simulated annealing 기반 power scheduler을 사용한다. 목표에 가까운 시드에 더 많은 에너지를 할당한다. 이 방법은 효과적인 전략이지만 우선순위 지정 절차가 없다.

### 2.3.4. P4
*AFLGo*는 *AFL*의 2가지 non-deterministic mutation 전략(havoc, splice)을 사용한다.
이 두 전략은 유리할 수 있지만 기존의 seed를 파괴할 수 있다. 일부 취약점은 특별한 전제조건이 있어야 도달할 수 있다. 이러한 측면에서 target과 매우 가까운 seed를 임의로 변경하기 적응형 mutation 전략이 필요하다.

### 2.3.5. summary
P1 : 더 다양한 거리 정의가 필요하며 짧은 추적에 초점을 맞추어선 안된다.

P2 : 직접 호출과 간접 호출 모두 분석해야 하며 정적 거리 계산중 다양한 호출패턴을 구별해야한다.

P3 : 현재 power schduling에 대한 조정이 필요하다. distance-guided seed prioritization이 필요하다.

P4 : adaptive mutation stategy가 필요하며 seed와 target의 거리에 따라 세말한 변형과 거친 변형을 최적으로 적용해야한다.

# 3. Approach Overview
*Hwakeye*`s workflow
![figure3]()

## 3.1. Static Analysis
먼저 inclusion-base pointer를 기반으로 CG를 정확하게 구성하여 가능한 모든 호출을 포함한다. 또한 각 함수에 대한 CFG를 구성한다.

이후 CG와 CFG를 기반으로 *Hawkeye*의 direction을 높이는 여러 유틸리티를 개선한다. 유틸리티는 다음과 같다.
1. Function level distance
    
    [Adjacent-Function Distance Augmentation](#42-adjacent-function-distance-augmentation)를 기반으로 CG를 통해 계산된다. 이 거리는 BB-level distance를 계산하는데 사용되며 fuzzing중 [Covered Function Similarity](#442-covered-function-similarity)를 계산하는데도 사용된다.
2. BB Level distance
   
   CG, CFG를 기반으로 계산되며 target site중 하나에 도달할 수 있다고 간주되는 BB에 대해 정적으로 계측된다. [Basic block trace distance](#441-basic-block-trace-distance) 계산에 사용된다.
3. Target function trace closure

    CG에 따라 각 target site에 대해 계산되며 target site에 도달할 수 있는 함수를 얻는다. fuzzing중 [Covered Function Similarity](#442-covered-function-similarity)를 계산하는데도 사용된다.

마지막으로 프로그램은 edge transitions (*AFL*과 유사), accumulated BB trace distance (*AFLGo*와 유사), covered function을 추적하기위해 계측된다.

## 3.2. Fuzzing Loop
fuzzer는 piority seed queue 에서 seed를 선택한다. target site에 "가까운" seed에 더 많은 변형 기회(에너지)를 주는것을 목표로 하는 power scheduling을 적용한다.

*coverd function similarity*와 *BB trace distance*의 조합에 따라 결정된다. 변형중 새롭게 생성된 각 test seed에 대해 trace를 얻어 3.1절의 유틸리티를 기반으로 *coverd function similarity*와 *BB trace distance*를 계산한다.  

에너지가 결정된 후 seed에 따라 [Adaptive Mutation](#45-adaptive-mutation)을 수행한다. 이후 더 많은 에너지를 가지거나 target function에 도달한 새로 생성된 seed를 우선순위에 두어 평가한다. ([seed Prioritization](#46-seed-prioritization))
# 4. Methodology


## 4.1. Graph Construction
선택된 seed와 target site까지의 거리를 계산하기 위하여 CG,CFG를 구축한다음 이를 결합하여 final inter-procedural CFG를 구성한다. CG는 [4.2](#42-adjacent-function-distance-augmentation), [4.3](#43-directedness-utility-computation)에서 function level distance를 즉정하는데 사용된다. CFG(inter-precedural CFG)는 [4.3](#43-directedness-utility-computation)에서 BB level distance를 즉정하는데 사용된다.

- `inclusion-based pointer analysis`를 사용

이 알고리즘은 "p:=q" 형태의 문장을 "q의 points-to set 은 p의 points-to set의 부분집합이다" 라고 번역하는 것이다.

points-to 의 전파는 4개의 규칙으로(address-of, copy, assign, dereference) 적용된다. 이 분석은 context-insensitive, flow-insensitive한데 함수의 호출 문맥과 내부 문장 순서를 모두 무시한다.

결국 프로그램의 모든 지점에 대해 유효한 단일 points-to set가 계산되어 모든 가능한 직,간접 호출을 포함하는 정밀한 CG를 만든다.

$O(n^3)$의 시간복잡도를 갖는다. 이것은 LLVM의 내장 api보다 더 정밀하다.

각 함수의 CFG는 LLVM의 IR을 기반으로 생성된다. inter-procedure flow graph는 CG,CFG의 call site를 수집하여 구성된다. 이러한 정적 분석을 통하여 P2가 달성된다. 
## 4.2. Adjacent-Function Distance Augmentation
P1을 달성하기 위해 CG를 기반으로 (immediate) 호출 관계 패턴을 고려하는 정적 분석을 제안한다.

![figure4]()


위의 그림에서 여러가지 호출패턴이 존재한다. Fig. 4a 에서는 fa, fb 모두 존재한다. 그러나 fc는 호출되지 않을 수 있다. 확률적인 관점에서 두 경우 모두 fa-fb가 fa-fc보다 가까워야 하며 4a에서의 fa-fb가 4b에서보다 가까워야 한다. E따라서 caller, callee간의 즉시 호출 관계로 정의된 거리를 증가시키기 위하여 두가지 metric을 제안한다.

1. caller에 대한 callee의 call site 발생 횟수 $C_N$을 정의하고 이 효과를 나타내기 위하여 $ϕ(C_N)$=$ϕC_N+1\over ϕC_N$ factor를 적용한다. ϕ는 보통 2인 상수값이다.
2. caller 내의 callee의 call site가 적어도 하나 있는 BB의 수 $C_B$ 이를 나타내는 $ψ(C_B)$=$ψC_B+1\over ψC_B$를 정의한다. ψ는 보통 2인 상수값이다. 

두 factor 함수 모두 단주 감소 함수이며 $C_N, C_B -> ∞$일때 $ϕ, ψ$ 모두 1로 수렴한다. 이로 이루어진 최종 adjustment factor는 ϕ와 ψ의 곱으로 이루어져 있을 것이며 다음과 같이 `agumented adjacent-function distance`가 정의된다.

![expression1]()

위의 4a에 적용해 보자면 다음과 같다.
    
> $C_N(f_a,f_b)=2, C_B(f_a,f_b)=2, d'_f(f_a,f_b)=1.25*1.25=1.56$ 

> $C_N(f_c,f_c)=1, C_B(f_c,f_c)=1, d'_f(f_a,f_c)=1.5*1.5=2.25$ 

만약 loop가 존재한다면 runtime에서 함수가 여러번 호출될 수 있다. 그러난 다른 seed로 실행될때 몇번 실행될찌 불확실하다. 하지만 실제 실행에서 loop내의 call site에서 호출은 일반적으로 비슷한 효과를 갖는다. 따라서 루프 내부의 함수 호출은 많은 실행 trace 다양성을 가져오지 않는다.

적용된 방식은 정적분석과 효율성 사이에 절충을 목표로 한다. 

## 4.3. Directedness Utility Computation
[4.2](#42-adjacent-function-distance-augmentation)에서 `augmented function distance`는 호출 패턴에 따라 계선된다. CG에서 간선의 가중치로 이를 할당하여 *Dijkstra shortest path algorithm*을 사용하여 두 함수의 function-level distance를 계산할 수 있다. 이를 이용하여 BB-level distance를 유도할 수 있다. 또한 [coverd function similarity](#442-covered-function-similarity)의 계산에도 사용된다.

### 4.3.1. Function Level Distance
CG를 이용하여 계산되며 현재 함수에서 target 함수까지의 평균 거리를 나타낸다. 다음과 같이 정의된다. CG위에서 n에서 target function $t_f$까지의 djikstra shotest path based on augmented function distance 이다.

![expression2]()

### 4.3.2. Basic Block Level Distance
BB level distance $d^o_b(m_1,m_2)$는 CFG G(n)에서 $m_1, m_2$까지의 edge의 최소 개수로 정의된다. 

BB m 에서 호출되는 함수의 집합을 $C_f(m)$라고 하면 $C^T_f=\set{n|R(n,T_f) \not ={∅ }, n ∈C_f(m)}$ 이다.

> n -> $T_f$,  m -> n 경로가 존재하는 함수 n의 집합

$Trans_b=\set{m|∃G(n),m ∈ G(n),n ∈ F,C^T_f(m) \not = {∅}}$

> $C^T_f$에 존재하는 함수 m의 집합

주어진 BB m에 대해 target BB $T_b$의 거리는 다음과 같이 정의된다.


![expression3]()

`expression2,3`은 *AFLGo*와 동일하다. 하지만 $d_f(n,t_f)$의 정의가 다르다.

### 4.3.3. Target Function Trace Closure
$ξ_f(T_f)$는 main에서 target function에서 도달할 수 있는 trace를 의미한다.

> 만약 경로가 여러개면 어떻게 되는거지?
## 4.4. Power Scheduling
dynamic fuzzing중 2가지 metric을 적용하여 power scheduling을 수행한다.
### 4.4.1. Basic Block Trace Distance
seed s 에서 target BB $T_b$까지의 거리는 다음과 같이 정의된다.

![expression4]()

*AFLGo*와 동일하게 s의 실행 trace에 있는 모든 BB에 대해 $T_b$ 까지의 BB-level distance의 평균을 계산한다. 또한 이를 정규화 하여 최종 거리를 얻는다.

> $\bar{d}_s(s,T_b)=$$d_s(s,T_b)-minD\over maxD-minD$

### 4.4.2. Covered Function Similarity
seed execution trace와 function level target execution trace 간의 유사성을 측정한다. BB를 사용하지 않는 이유는 상당한 overhead를 도입하기 때문이다.

유사성은 "expected trace"에서 더 많은 함수를 커버하는 seed가 변형될 기회가 더 많다는 직관을 기반으로 계산된다.

![expression5]()


### 4.4.3. Scheduling
seed가 실행하는 trace가 프로그램 내의 target site에 도달할 수 있는 예상 trace중 하나에 더 가까운 경우 더 많은 변형을 생성해야 한다.

trace distance에 기반한 scheduling은 특정 trace pettorn을 선호할 수 있다. *AFLGo*처럼 target site에 도달할 수 있는 긴 경로가 소외될 수 있기 때문에 이를 완화 하기 위해 trace distance와 trace simlarities를 모두 고려하는 power function을 제안한다.

![expression6]()

$c_s와 d_s$이는 다음과 가은 차이가 존재한다.

1. $d_s$는 CG, CFG를 모두 고려하지만 $c_s$는 CG만 고려한다. ($d_s$도 주요 효과는 CG)
2. $d_s$는 target으로 가지 않는 trace를 처벌하지 않지만 $c_s$는 처벌한다.
3. traget에 도달하는 여러 trace가 주어진 경우 $d_s$는 짧은 trace를, $c_s$는 예상 trace와 중복되는 trace를 선호한다.

이러한 의미로 성질이 다른 두 metric을 이용하여 균형을 추구한다. 물론 여전히 편향이 존재할 수 있다. 저자의 의견은 문제가 아님 적절한 mutation을 사용하면 됨


## 4.5. Adaptive Mutation
GF의 mutation은 두가지가 있다.

- coarse-grained : bulks of byte를 변형
- fine-grained : few byte-level mutation

coarse-grained 에 대해서 다음과 같은 사항을 고려해야한다.

1. Mixed havoc : 바이트 덩어리 삭제, 주어진 덩어리를 버퍼의 다른 바이트로 덮어쓰기, 특정 라인 삭제, 복제등 여러 거대한 변형
2. Semantic mutation : 프로그램 input이 JS, XML, CSS등 semantic relevant input file인 경우 *Skyfire*를 따르며 랜덤 AST 위치에 다른 하위 tree를 삽입, 주어진 AST 삭제, 주어진 위치를 다른 AST로 대체
3. Splice : queue의 두 seed를 혼합한 후 Mixed havoc

![algorithm 1]()

위 알고리즘은 주어진 seed s에 대한 우리의 adaptive mutation의 workflow를 보여준다.

line 10 : s가 target function에 도달할때 coarse-gained mutation을 줄인다.

![algorithm 2]()

위 알고리즘은 seed에 대한 coarse-grained mutation이다. 우리는 상수에 경험값을 할당한다.

> γ = 0.1, δ = 0.4, σ = 0.2, ζ = 0.8

## 4.6. Seed Prioritization

- priority queue를 사용하면 좋지만 insertion에 O(log n)이 소모됨 이를 해결하기 위해 queue를 3단계로 나눔
- priority queue를 모방하였지만 상수 시간복잡도를 갖음 

![algorithm3]()

새로운 시드에 우선순위를 부여하는 아이디어는 다음과 같다.
1) 새로운 trace를 cover : 초기 seed가 target과 멀리 떨어져 있을때 필요
2) target seed와 큰 유사성
2) target function을 cover

2,3은 DF에 특화되어 있다. 

*AFL*의 loop bucket approach를 참고하여 새로운 coverage를 제공하지 않는 대량의 "equivalent" seed를 필터링한다. 이후 남은 시드에 이 전략을 사용한다.

# 5. Evaluation

- *AFL's LLVM mode*위에 static instrumentation 구현
- *SVF* (static value-flow analysis tool)을 이용한 pointer analysis
- *AFL*위에 *Rust*를 사용한 동적 fuzzer 구현
- fork server, shared meemory base BB edge tracking, (non)deterministic mutator등은 AFL을 따름
- 성능을 희생하지 않고 모듈화, 확장성을 고려하여 설계함

*Hawkeye*의 instrumentation 세 부분으로 구성됨
1. execution trace를 tracking하는 BB ID
2. BB trace distance를 결정하는 BB distance 정보
3. 함수를 추적하는 function ID

## 5.1. Evalution Setup
RQ1. 정적 분석이 가치 있는가
RQ2. crash reproduction performance
RQ3. dynamic stategies의 성능 
RQ4. 특정 target site에 도달하는 능력은 좋은가

평가 데이터셋 : *GNU Binutils* *MJS* *Oniguruma* *Fuzzer Test suite*

Evalution Tools : *AFL* *AFLGo* *HE-Go*

## 5.2. Static Analysis Statistics
![table1]()
- 표에서 ics/cs 는 호출 site중 간접 호출의 비율

간접 호출 비율이 최대 44%로 정확한 CG를 구축하는것의 중요성을 보여줌. 또한 CN, CB>1인 경우도 많기에 호출 관계의 다양한 패턴을 고려하는것이 중요함

정적 계산의 overhead와 관련하여 *cxxfilt*를 제외하면 몇초안애 생성 가능, *cxxfilt*은 복잡한 프로젝트임, *SVF*에 구현된 poiter analysis로 인한 병목 발생

source-code가 변경되지 않는한 CG는 재사용 가능
## 5.3. Crash Exposure Capability
## 5.4. Target Site Covering
## 5.5. Answers to Research Questions
RQ1. 정적 분석 시간의 비용은 runtime 에 비해 수용가능하다. 대부분의 경우 *AFL*을 능가한다.

RQ2. crash detecting에 매우 우수한 성능을 보임. 다른 도구보다 더 빠르게 감지할 수 있으며 여러번 실행 속에서도 결과는 안정적이다.

RQ3. 동적 전략은 효과적이다. *Hawkeye*의 방법은 *AFLGo*의 simultated annealing 보다 효과적이다.

RQ4. target site에 도달할 수 있는 능력이 잇다고 확신한다.

## 5.6. Threats to Validity
### 5.6.1. internal threats
1. 구현 과정속 (algorithm 1,2)는 사전에 정의된 값을 사용하여 결정을 내린다. (γ=0.1, δ=0.4, σ=0.2, ζ=0.8) 는 예비 실험과 경험에 따라 구성된다. 이러한 임계값의 영향을 조사하고 최적의 구성을 찾아내기위한 연구가 필요하다.

2. *LLVM*, *SVF*와 같은 lightweight program analysis tool에 의존하여 거리를 계산하기 때문에 이 도구의 문제가 최종 결과에 영햐응ㄹ 줄 수 있다. *Hawkeye*는 모듈화 되어 있고 다른 static analysis tool과 쉽계 통합될 수 있기에 다른 도구를 사용할 수 있다.

### 5.6.2. external threats
dataset과 CVE의 선택에서 발생한다. 우리는 *AFLGo*에서 사용된  *Binutils* 사용하였찌만 더 많은 프로젝트에 대한 연구로 일반화 되어야 한다.

# 6. Related Work
## 6.1. Directed Grey-box Fuzzing
## 6.2. Directed Symbolic Execution
## 6.3. Taint Analysis Aided Fuzzing
## 6.4. Coverage-based Grey-box Fuzzingb
# 7. Conclusions
- DGF *Hawkeye*를 제안함
- DGF를 위한 4가지 바람직한 속성을 포함함
- see 우선순위 지정, power scheduling, mutation strategies를 사용하여 target site에 신속하게 도달할 수 있음 
# 8. Memo
동적 분석에서 닿지 않은 곳은 파악할 수 없으니까 정적 분석을 같이 사용한다.

논문 내용
- ξ가 여러개아닌가? similarity 계산 어떻게?
- 길만 따로 찾고 그 길을 따라 fuzzing 가능?
  
