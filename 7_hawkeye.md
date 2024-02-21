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

## 2.3. AFLGo’s Solution

# 3. Approach Overview
## 3.1. Static Analysis
## 3.2. Fuzzing Loop


# 4. Methodology
## 4.1. Graph Construction
## 4.2. Adjacent-Function Distance Augmentation
## 4.3. Directedness Utility Computation
## 4.4. Power Scheduling
## 4.5. Adaptive Mutation
## 4.6. Seed Prioritization
# 5. Evaluation
# 6. Related Work
# 7. Conclusions

# 8. Memo
동적 분석에서 닿지 않은 곳은 파악할 수 없으니까 정적 분석을 같이 사용한다.




길만 따로 찾고 그 길을 따라 fuzzing 가능?