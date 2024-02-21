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

각 함수의 CFG는 LLVM의 IR을 기반을 ㅗ생성된다. inter-procedure flow graph는 CG,CFG의 call site를 수집하여 구성된다. 이러한 정적 분석을 통하여 P2가 달성된다.
## 4.2. Adjacent-Function Distance Augmentation
## 4.3. Directedness Utility Computation
## 4.4. Power Scheduling
### 4.4.1. Basic Block Trace Distance
### 4.4.2. Covered Function Similarity
## 4.5. Adaptive Mutation
## 4.6. Seed Prioritization
# 5. Evaluation
# 6. Related Work
# 7. Conclusions

# 8. Memo
동적 분석에서 닿지 않은 곳은 파악할 수 없으니까 정적 분석을 같이 사용한다.




길만 따로 찾고 그 길을 따라 fuzzing 가능?