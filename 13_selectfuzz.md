[SELECTFUZZ: Efficient Directed Fuzzing with
Selective Path Exploration](https://seclab.cse.cuhk.edu.hk/papers/sp23_selectfuzz.pdf)

# 0. Abstract
- DGF는 새로 발견한 코드가 target과 관련이 있는지 여부와 관련없이 새로운 코드를 발견하는 input을 선호
- 이는 target code 에 지향하는데 도움이 되지 않음
- *SELECTFUZZ* : target과 관련있는 path를 선택적으로 탐색함, relevant code(path divergent code, data-dependent code)를 식별
- control, data dependency를 포착하여 relevant code block을 선택적으로 instrument, explore 함
- 새로운 distance matric 제시
# 1. Introduction
- 일반적인 DGF는 path 탐색에 있어서 AFL의 전략 (code coverage feedback)을 따름 > 관련 없는 코드를 explore 할때가 많음
- 도달 가능한 코드중 87%는 target code와 관련 없음 >  관련 없는 code를 explore 하지 않는 방식으로 DGF 기술을 개선
> Challenge
1. relevant code를 정확하게 식별할 것인가?
- path divergent code : target에 효율적으로 도달하기 위한 feedback (control flow condition) 
- data dependent code : 다양한 data로 code를 explore하도록 guide (data flow condition)

2. irrelevant code를 어떻게 제외할 것인가? 
- relevant code에 대해서만 instrument하여 feedback을 받음

> selectfuzz

- static analysis로 relevant code 식별
- path divergent code : path의 reachibility를 알아야함 > 이를 추론하기 위한 새로운 distance metric 제시
- distance metric으로 seed prioritization 수행

> contribute
- relevant code 개념 재시
- selective path explore를 적용한 DGF *SELECTFUZZ* 제시 > 기존 접근법 보완 가능
- 14개의 새로운 취약점 탐지
# 2. Background
> DGF를 개선하는 두가지 방법
1. SE, TA를 이용항 고품질 input 생성

2. fuzzer가 흥미로운 input을 식별 (이 논문의 중점)
- 흥미롭지 않은 input을 과도하게 testing 하는것은 낭비 > 보통 distance based, input reachability analysis 사용
## 2.1. Distance-based Input Prioritization
- 두 BB간의 거리는 CFG에서 dijkstra algorithm > input distance는 input이 방문한 모든 BB의 average distance 사용
- 이러한 input prioritization에 heuristic을 사용하며 항상 성능을 개선한다는 보장이 
- 여러 개선이 있지만 효율성을 크게 개선하지는 못하였음
## 2.2. Input Reachability Analysis
- rechability analysis는 최근 두가지 접근 방법이 있음
### 2.2.1. Deep Learning
- *FuzzGuard*는 unrechable input을 제거하는 DL 모델 훈련 > target에서 pre-dominating node를 미리 식별하고 이러한 node를 통과할 수 없는 input을 걸러내는것 > 즉 이전 실행에서 학습하여 pre-dominating node를 통과할 수 있는지 예측하는 모델
### 2.2.2. Path Pruning
- *Beacon*은 unrechable path에 실행을 조기에 종료 시킴 > 변수 정의와 branch에 checkpoint를 삽입하여 target에 도달하는 precondition을 추론하기 위한 interval analysis 수행 > precondition이 충족되도록 assertion 
- 도달하지만 irrelevant code를 실행하여 에너지 낭비

# 3. Problem statement
## 3.1. Relevant Code
- relevant code : target에 도달(control flow condition), vulnerability에 사용 (data flow condition) 하는 코드
- 이는 target에 접근하는지 분석하는 것과 다름 > 유일한 sucessor 라면 실행에 더 가까이 도달하는데 도움이 되지 않음

![listing1](./image/13_listing1.png)

![figure1](./image/13_figure1.png)

- 14~15 에서 rechability를 결정 할 수 있음
- control flow condition을 고려하면 항상 line 11에 도달할 수 있기 때문에 2~10 line의 code는 관련이 없음
- 11~13중 일부 rechable code가 target과 간접적인 control flow dependency를 갖고 있더라도 이곳에 instrumentation 하는 것은 non trivial한 부작용을 제공
- target reachalbe과 관련 없는 code block을 모두 제외한 다면 line 5와 같이 target에 사용되는 변수의 값을 변경하는 code를 제외하게 됨 > 우리는 line 5와 같이 data dependence한 코드를 relevant code에 포함한다.

## 3.2. Limitations of Existing Approaches
- 기존 DGF가 irrelevant code를 탐색하여 효율적이지 않음
- figure 1 : `a->c->e->h` 가정 > 우리는 f,b 를 실행하는 input을 relevant input으로 간주

> DGF가 irrelevant code를 탐색하여 효율성이 낮아지는 이유
1. irrelevant input에 대한 변형은 fuzzer가 계속 irrelevant input을 생성할 가능성이 높음
2. target에 도달하더라도 효율적으로 vulnerability를 trigger하기 어려움 > irrelevant code는 data dependency가 없기 때문
## 3.3. Research Goals and Challenges
- relevant code에 초점을 맞춘 더 나은 path exploreation 전략을 개발하고자함
> challenge
1. relevant code를 정확하게 결정하는 것
2. irrelevant code를 explore 하지 않는 방법
- 조기 종료와 같은 방법은 효율적이지 않음
- 관련 없는 코드를 삭제하는ㄴ 것은 오류를 일으킬 수 있고 유효한 PoC가 아닐 수 있음

# 4. SelectFuzz
![figure2](./image/13_figure2.png)

- relevant code만 식별하고 탐색함
- 새로운 distance metric을 활용하여 모든 rechable code를 식별, input prioritization에 사용됨
- inter procedural control flow, data flow analysis를 통하여 relevant code를 식별하고 선택적으로 instrumentation을 하여 code coverage feedback 제공
## 4.1. Block Distance
- BB에서 target까지 multi path reching "probability"를 개발함 > input distance는 input이 explore path에 있는 모든 relevant BB의 inter-procedural bb distance 에서 계산됨
> 장점
1. 모든 가능한 path를 고려하여 BB에서 target에 도달할 확률을 평가 가능
2. inter procedural analysis, call target analysis를 통하여 cross-fuction distance 측정
### 4.1.1. Block Distance
![algorithm1](./image/13_algorithm1.png)

- BB에 3가지 labeling ( 0 : initial, 1 : distance 계산중, @: distance 계산 완료)
- cal_prob()에서 reaching probability $P_b$ : b에서 target에 도달할 확률 추정
- $P_b$ : 모든 sucessor block의 reching probability의 평균으로 계산
- target이 여러개라면 어떤 target에든 도달할 확률로 계산
- loop, recursion의 경우 unroll 처리하여 한번만 계산
- cal_dist()에서 $P_b$의 역수를 block distance로 $d_{bb}(b,T)$를 계산
- cross function block distance를 계산하기 위하여 call site에 callee function을 sucessor로 추가함
- indirection call의 경우 target function을 추론하여 indirection call site 가능한 모든 callee를 연결함 > 이는 false positive error를 도입할 수 도 있지만 이런 경우 distacne가 매우 커서 선호되지 않을 것이다 /// 이해필요

### 4.1.2. Input Distance
- distance matric에서 input distance는 cover된 모든 BB의 최단 block distance (다른 것은 평균을 사용함)
- 최단 거리가 상황을 잘 반영하는지 보여줄 수 있다.
### 4.1.3. Example
![listing2](./image/13_listing2.png)'
![figure3](./image/13_figure3.png)

$SelcF : d_{bb}(a,T) = 2, d_{bb}(h,T)=2, d_{bb}(h,T)=4$ /// 해설 필요

## 4.2. Selective Path Exploration
- relevant code를 선택적으로 instrumentation 하고 explore함
### 4.2.1. Relevant Code Identification
> path divergent code : rechable path와 unrechable path의 교차 지점
- target을 향하는 input을 구별할 수 있는 중요한 feedback을 제공
- 따라서 intrumentation 되지 않고 coverage나 distance feedback을 제공하지 않는 irrelevant code는 인식하지 못함

![listing1](./image/13_listing1.png)
- line 14, 15의 path divergent code는 line 13에서 생략된 code block의 data에 의존적일 수 있음
- 하지만 이를 무시함으로 더 작은 explore space를 갖을 수 있다. 그리고 line 13 에서 y,z값을 변경 하는것은 fuzzer에 의해 방문될 것이다. > WindRanger와 비슷함

> data dependency code : target에서 사용되는 중요 변수에 영향을 미치는 code
- target에 도달하는데 도움이 되지 않지만 exploitation 단계에서 도움이 될 수 있음

![figure1](./image/13_figure1.png)
- e, f : path divergent code, b, g : data dependent code

### 4.2.2. Input Prioritization and Power Scheduling
- intrumentation을 다르게 했기 때문에 input prioritization과 power scheduling을 변경해야함

#### 4.2.2.1. Input Prioritization
- relevent BB 간의 새로운 edge를 cover하거나 기존 edge를 new scale로 증가시키는 input을 선택
- selectfuzz가 항상 relevant input을 만들 수 있는것은 아님 > 이럴 경우 더 작은 input distnace를 갖는 irrelevant input을 변형함


#### 4.2.2.2. Power Scheduling
- AFLGo의 annealing-based power scheduling을 사용함
- [4.2.1](#421-relevant-code-identification)에 의하면 irrelevant BB *irbb* 가 
# 5. Impelementation
# 6. Evaluation

## 6.1. Triggering Known Vulnerabilities (RQ1)
## 6.2. Ablation Study (RQ2)
## 6.3.  Understanding Performance Boost (RQ3)
## 6.4. Benchmarking (RQ4)
## 6.5. Detecting New Vulnerabilities (RQ5)
# 7. Discussion
## 7.1. False Positives in Relevant Code Identification
## 7.2. Solving Complex Path Constraints
## 7.3. Identifying Vulnerable Code Paths
# 8. Related Work
## 8.1. Improving Directed Fuzzing Efficiency
## 8.2. Targeted Program Analysis
# 9. Conclusion
