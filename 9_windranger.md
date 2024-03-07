[WindRanger: A Directed Greybox Fuzzer driven by Deviation
Basic Blocks](https://dl.acm.org/doi/pdf/10.1145/3510003.3510197)

# 0. Abstract
- DGF는 execution trace가 target site와 가까운 seed를 우선시하여 direction을 얻음
- 따라서 distance를 구하는것이 중요함
- *AFLGo*는 정적 분석중 BB-level distance를 계산하여 seed 거리를 계산함.
- 모든 BB가 동일한 중요성을 갇는것이 아니며 execution trace가 target에서 벗어나기 시작하는 특정 BB(deviation BB)들이 있음

- 이 논문에서는 deviation BB를 활용하여 *Wind Ranger* 제안
- deviation BB를 식별하기 위해 static reachability analysis, dynamic filtering을 사용
- deviation BB와 관련된 data flow information을 통하여 seed distance 계산, mutation, seed priority 지정, explore-exploit scheduling을 수행

# 1. Introduction
- GF는 미리 정의된 target에 도달하는 direction이 부족하다.
- DGF는 CGF보다 빠르게 PUT의 target site에 도달할 수 있다.
- patch testing, crash reproduce, static analysis report verification와 같은 특정 시나리오에 사용됨


*AFLGo*는 정적분석중 CG, CFG의 각 BB와 target site 사이 거리를 계산함, 실행중 각 BB의 거리 값을 집계하여 seed의 거리를 계산


*AFLGo*의 seed distance의 두가지 함정
1. execution trace의 모든 BB를 사용하여 계산된 seed distance는 대상 site로 실행을 주도하는 모든 block이 핵심이 아니기에 편향될 수 있음 (교수님의 루프 그림)
2. seed distance 계산은 control flow information에 의해 기반하여(CG, CFG 상의 edge의 수) data flow information은 무시됨, 하지만 이 거리는 target에 도달하는 난이도를 대표할 수 없음, 예를 들어 checksum을 확인하는 branch의 경우 valid input에 대한 edge가 더 어렵기 때문에 거리 계산에 다르게 취급해야함.

- target site에 도달하지 못한 execution trace를 검토하여 target site에서 벗어나기 시작하는 key BB들이 존재함. 이를 DBBs(drivation basic blocks)라고 함

- DBBs에서 execution tarce를 변경할 수 있다면 target site에 도달할 수 있음


이 개념을 도임한 *WindRanger*는 다음과 같은 흐름을 갖는다.
1. 정적분석중 target site에서 벗어날 수 있는 분기를 포함하는 잠재적 DBBs를 식별한다. 이후 식별된 BB와 target site까지 거리를 계산한다.
2. fuzzing중 seed와 execution trace를 이용하여 DBBs를 선택한다.
3. DBBs를 기반으로 seed distance를 계산하고 DBBS의 condition matching difficulty를 기반으로 distance를 조정한다.
4. distance를 기반으로 power scheduling strategy를 조절하여 seed priority를 정한다.

이 외에도 DBBs를 바탕으로 seed priority strategy, mutation strategy, explore-exploit switching decision process등을 개선한다.

contribution
- DBBs의 개념과 static, dynamic analysis를 사용하여 이를 식별하는 접근 법 제시
- DBBs를 이용한 seed scheduling, mutation strategy를 사용한 DGF *WindRanger* 제시
- *WindRanger*를 평가하여 우수성을 입증


# 2. Motivating Example
![figure1]()

`CVE-2018-8962` : `decompileJUMP`함수에서 `getName` 함수를 호출할때 UAF가 발생

DGF를 사용한다면 16,22줄을 포함하는 BB를 target site로 설정하고 각 BB와 target site 거리를 계산해야함.

X:Y 는 line X 에서 taret에 도달하기위해 Y개의 edge를 지난다는 의미

## 2.1. Limitations of Existing Approaches
DGF가 A,B,C를 찾았고 해당 실행 trace는 figure 1에 있음 (실행 경로는 점선) seed distance를 구해 본다면 다음과 같다.
> A = 3+5+4+3+2+2+1 / 7 = 2.86
> 
> B = 3+2+2 / 3 = 2.33
>
> C = 3+2+1 / 3 = 2

1. 이를 바탕으로 seed C가 선호되어야 한다. 하지만 실제 15줄의 조건은 fuzzer가 뚫기 매우 어려움, 결과적으로 path condition을 고려할때 C는 좋은 선택이 아님 
 
> 즉 control flow information만 사용하여 seed를 계산하는것은 충분하지 않고 data flow information 을 고려해야함

3. A,B 중 B가 더 가까우므로 B를 선호 하지만 A는 21:1을 실행하여 더 좋은 seed 이지만 7:5, 9:4, 12:3을 실행하여 distance가 증가되었기 때문에 선호되지 않는다. 
> 이를 바탕으로 모든 BB를 기반으로 계산된 seed distance는 편향될 수 있으며 부정확 할 수 있음을 보여준다.
## 2.2. Our observations
target site에 도달하지 못하는 trace는 특정 BB에서 벗어나게됨

A : 21:1 ,B : 20:2, C : 15:1 에서 벗어남

이러한 BB를 DBB라고 부름
## 2.3. Our Approach
DBB만의 평균으로 거리를 계산한다면 시드들을 1,2,1의 거리를 갖게 된다, 여기서 어려운 조건인 C에 대한 거리를 추가 조정할 수 있다면 A에 우선순위를 둘 수 있다.

- 이 예시에서 DBB가 가장 짧은 거리를 가진 BB라는것은 우연히 일치함 
- seed D 가 <6:3->7:5->9:4->10>인 경우 6:3이 9:4 보다 더 짧음에도 9:4에서 target site에서 벗어나므로 9:4가 DBB가 된다.

*WindRanger*는 DGF의 맥락에서 DBB의 명확한 정의를 제공하고 execution trace에서 그들을 식별하기 위한 접근방식을 제안함.
또한 더 나은 seed distance 계산을 위해 DBB 정보 및 data flow information을 활용하는 접근방식을 제안함. 
# 3. Approach of WindRanger
![Figure2]()

- *WindRanger*는 정적 분석을 통해 DBB의 가능성이 있는 후보를 식별하고 distance를 계산
- CGF와 같이 Graybox exploration을 위한 edge coverage를 수집

fuzzing 과정에서 seed input을 선택하기 전에 exploration or exploitation에서 실행할지 결정 - local optima에 빠지지 않도록 동적으로 결정됨 (exploration 에서는 일반 CGF 처럼, explotation 에서는 distance가 짧은 seed에 우선 순위)

queue에 seed가 선택되고 power schedule이 할당 된 후 data flow analysis 결과와 함께 mutation 수행

mutation input이 실행된 후 execution trace를 분석하여 input을 새로운 seed로 유지할지 결정함.

새로운 seed로 유지되는 경우에 DBB를 확인하여 target site로 거리를 계산하고 data flow analysis 결과에 맞게 거리 값을 조정함

[1. 정적 및 동적 분석을 사용하여 DBB를 식별](#31-deviation-basic-blocks-identification)

[2. probing based taint analysis 수행](#32-probing-based-taint-analysis)

[3. seed 거리를 계산하고 taint analysis 결과로 미세 조정하는 방법](#33-seed-distance-calculation)

[4. taint analysis 결과로 input을 mutation 하는 방법](#34-data-flow-sensitive-mutation)

[5. DBB를 사용하여 seed priority를 정하는 방법](#35-seed-prioritization)

[6. explore, exploit witch decision](#36-dynamic-switch-between-explore-and-exploit-stage)

## 3.1. Deviation Basic Blocks Identification
- 정적 분석을 통해 잠재적 DBB를 찾음 이후 fuzzing중 실행된 BB를 분석하여 execution trace 내의 DBB를 식별함

### 3.1.1. static DBB collection

Definition 1

![expression1]()

    Φ(𝑃) : potential DBB
    ALLBB(P) : P의 모든 BB
    T : target site set
    isReachable(x,T) : inter-procedural CFG에서 x가 T에 있는 target site에 도달 할 수 있는지
    Successor(b) : b의 모든 successor BB set

Potential DBB는 적어도 하나의 target에 도달할 수 있고 그것의 successor중 하나는 target site에 도달할 수 없는 BB이다. 

실제로 *WindRanger*는 iCFG를 구성하고 각 BB와 target site의 도달 가능성을 계산한다. 

### 3.1.2. dynamic DBB collection
Definition 2

![expression2]()

    Φ(𝑃) : 주어진 potential DBB 
    𝜉(𝑠) : seed S의 execution trace의 모든 BB
    ReachableSucc(b) : target site에 도달할 수 있는 BB b의 모든 succesor

DBB의 모든 succesor중 target site에 도달할 수 있는 것은 execution trace에 존재해선 안된다.


## 3.2. Probing-based Taint Analysis
- *WindRnager*는 *FairFuzz*와 *GreyOne*과 유사한 probing-based taint analysis를 사용함
- 주어진 branch의 constraint에 영향을 줄 수 있는 seed의 byte에 대한 data flow information을 수집한다.
- 이 정보는 *effector map*이라고 불리는 hash map에 저장된다. (key : BB address, value : branch constraint에 영향을 주는 seed  byte index set) 이 정보는 3.3, 3.4에서 사용된다.

![algorithm1]()

위 알고리즘은 *effector map*을 생성하는 과정이다.

- `ValueOf(var,s)` : seed `s` 에 대한 variable `var`의 값을 반환
- `OP`  : bit flip, insertion, deletion

seed `s`와 execution trace `𝜉(𝑠)`에 대해 

1. `𝜉(𝑠)`에서 branch constraint와 관련된 모든 변수 `VAR`을 추출
2. 정의된 muatation operation `OP` set을 사용하여 seed를 byte by byte로 mutation함
3. mutated input에 대해 `VAR`의 값이 변하는지 확인
4. 값이 변경된 경우 *effector map*을 업데이트 하여 `s`에서 변형된 위치가 `VAR`에 영향을 줄 수 있음을 기록

## 3.3. Seed Distance Calculation
![expression3]()


    $d_b(m,T_b)$ : target BB와 DBB(m) 사이의 거리
    Φ(s) : seed s 의 execution trace 내의 DBB

- 정의는 *AFLGo*와 *HawkEye*와 동일하지만 모든 BB가 아닌 DBB를 사용함
- control flow information만 사용하였기에 정확하지 않음
- DBB의 distance를 data flow information으로 다음과 같이 조정

![expression4]()

![expression5]()

    Ψ(𝑠,𝑚) : DBB의 constraint를 해결하는 난이도
    NumOfEffectiveByte(s,m) : s의 *effector map*에 따라 m의 constraint에 영향을 줄 수 있는 input byte의 수
    𝛾 : granularity를 조절하는 constant ratio

- 많은 input byte가 constraint에 영향을 미칠 수록 fuzzer가 이를 해결하기 힘들다

branch distance 대신 constraint에 영향을 미치는 byte 수를 사용하는 이유는 복잡한 데이터 변환을 거치는 경우 branch를 통과하는 난이도를 반영할 수 없기 때문이다.

실험에서 constraint에 영향을 미치는 byte 수를 사용하면 좋은 결과를 얻을 수 있다는 것을 경험적으로 발견하였다. [#4](#4-implementation-and-evalutation)

## 3.4. Data Flow Sensitive Mutation
*Effector map*에서 제공하는 정보를 바탕으로 data flow sensetive mutation을 사용한다.

seed와 *effector map*이 주어진 경우 exploration or exploitation stage 인지 확인한다.

만약 exploitation stage의 경우 DBB의 constraint와 관련된 input byte를 high priority byte로 생각한다.

exploration stage의 경우 모든 constrait와 관련된 byte를 high priority byte로 식별한다.

또한 constraint 와 관련된 input byte에 대해 이 byte가 연속적인 경우 변수와 관련된 byte가 동일한 값을 공유하는지 확인한다. 만약 동일하다면  input byte가 데이터 변환을 거치지 않았을 가능성이 높기 때문에 mutation을 적용하기 전에 관련 input byte를 cconstraint comparison operand로 교체해보려고 시도한다.
## 3.5. Seed Prioritization
data flow sensitive mutation과 동일하게 seed prioritization은 exploration과 explotitation stage에서 다르게 작동한다.

### 3.5.1. explotitation stage
- 2-level priority queue를 유지한다. 
- DBB를 cover할 수 있는 seed 목록을 찾는다. seed distance에 따라  오름차순으로 정렬한다.
- 각 목록에서 첫번째 seed를 선택하여 더 높은 priority를 갖는 queue에 넣고 나머지 seed는 다른 큐에 넣는다.

mutation을 위한 다음 seed를 선택할때 더 높은 priority queue에서 선택할 가능성이 더 높다.  
### 3.5.2. exploration stage
일반 CGF와 동일하게 작동하여 더 높은 code coverage를 달성하는 seed를 우선시한다.

이외는 *AFLGo*에서 사용되는 simulated annealing algorithm을 사용한다. seed distance만 다르다.
## 3.6. Dynamic Switch between Explore and Exploit Stage
- DGF는 local optima에 갇히지 않기 위하여 충분한 coverage exploration이 필요
- 예를 들어 *AFLGo*에서는 user-defined time budget을 사용한다. 하지만 user가 프로그램에 적함한 time budget을 찾기 위해선 기반 지식이 필요이 문제를 해결하기위하여 
- DBB의 실행 상태에 따라 explore, exploit stage를 동적으로 전환한다.
- DBB를 global set으로 유지하여 exploit stage 에서 DBB가 충분히 exploite 되었을때 exploration stage로 전환된다.
- DBB가 충분히 exploite디었는지 확인하기 위하여 DBB가 실행된 횟수를 확인

![expression6]()

![expression7]()

![expression8]()

    hit (x,b) : input x가 BB b를 hit 할 수 있는지 여부
    numHit(b) : b가 hit된 횟수
    I : fuzzing으로 생성된 모든 input set
    T : fuzzing중 *WindRanger*가 도달한 모든 BB set
    v : constant factor

- exploration -> explotitation : 새로운 DBB를 발견
- 이 과정은 seed mutation을 마친 후에 새로운 seed를 선택하기 전에 발생
# 4. Implementation and Evalutation
- static analyzer : *LLVM IR*에서 iCFG를 구성하기위하여 *SVF* 사용 
- dynamic fuzzer : *AFL 2.52b* 기반

RQ1 : target site에 도달하는 능력

RQ2 : target bug를 reproduce 하는 능력

RQ3 : compnent가 성능에 어떤 영향을 미치는가

RQ4 : real-world bug hunting에 도움이 되는가
## 4.1. Evaluation Setup
### 4.2.1. Evaluation Datasets
- *UniBench*의 20개의 프로그램에서 각 4개의 target site를 지정하고 TTT(Time-to-Targets)을 측정함 (RQ1,RQ3)
- *AFLGo Test Suite*의 n-day vulnerabilities 프로그램 set 사용 (RQ2)
- *Fuzzer Test Suite* : *Google*의 fuzzer benchmark set 사용 (RQ2)
### 4.2.2. Evaluated Techniques
- *WindRanger*
- *AFLGo* : 대표적인 DGF
- *AFL* : 대표적인 CGF
- *FiarFuzz* : CGF 이지만 taint analysis를 사용
### 4.2.3. Evaluation Criteria
- TTT : target site에 도달하는 첫 input을 생성하는데 사용된 시간 (알려진 bug 없이 benchmark에서 기술을 평가할때)
- TTE : bug를 발생하는데 사용된 시간 (알려진 bug가 있는 benchmark에서 기술을 평가)
### 4.2.4. Experiments Settings
- *AFLGo Test Suite* : 8시간 20회
- 나머지 : 24시간 10회
- Mann-Whitney U test를 사용하여 통계적 유의성 측정
- Vargha-Delaney static를 사용하여 더 나은 성능을 보일 확률을 측정
## 4.2. Target Site Reaching Capability (RQ1)
- DGF의 TTT를 평가하기 위한 표준 dataset은 없음
- *Google fuzzer test suite*에는 주어진 target site에 3 프로젝트가 있음 하지만 양이 적고 일부 target site는 제공된 seed로 쉽게 도달할 수 있음
- 풍부하고 유효한 dataset을 얻기 위하여 *UniBench*를 사용함
  
![table1]()

## 4.3. Bug Reproducing Capability (RQ2)
baseline fuzzer와 *WindRanger*의 TTE를 비교함으로써 bug reproducing 능력을 연구함
### 4.3.1. AFLGo Test Suite
![table2]()
### 4.3.2. Fuzzer Test Suite
![table3]()
## 4.4. Impact of Different Components (RQ3)
- 각 component를 개별적으로 비활성하여 [#4.2](#42-target-site-reaching-capability-rq1)와 동일한 실험을 진행

![table4]()

data flow sensitive mutation > seed Prioritization > explore-exploite stage switch > distance calculation
## 4.5. New Vulnerability (RQ4)
- *WindRanger*의 prototype을 보안 전문가에게 제공
- 수동으로 code review를 하여 50개의 target 설정
- 그중 하나에서 취약점을 발견하여 CVE-ID 할당

해당 지점을 target site로 하여 *AFLGo*와 비교

![figure3]()
# 5. Threats to Validity
## 5.1. Internal threat
1. iCFG를 얻기위하여 *SVF*를 사용함 하지만 부정확한 pointer analysis로 인한 iCFG의 오류 가능성 > reachability analysis에 문제
2. configurable option (ex [EES의 variable](#36-dynamic-switch-between-explore-and-exploit-stage))
## 5.2. External threat
실험 설정과 관련하여 실험에서 무작위성을 완화 하기위하여 통계적 검정을 사용함. 다른 구현의 영향을 완화하기위하여 AFL 기반의 basline을 선택함, 또한 TTE 실험에 대한 세밀한 crash triage 필요
# 6. Related Work
## 6.1. Directed Symbolic Execution
- DSE는 target site에 도달하기위해 SE를 사용하는 기술
- DSE는 PUT의 원하는 부분을 효율적으로 테스트할 수 있지만 path-explosion으로 인해 실제프로그램에서 효과적이지 않음
- 우리의 DGF는 가벼운 program analysis만 사용하여 실제프로그램에서 더 효과적임
## 6.2. Directed Grey-box Fuzzing
- *AFLGo* : sedd trace와 target site간 거리를 계산하는 방법을 제안, 이 거리를 power scheduling에 사용
- *HawkEye* : trace agumented power schedule, seed prioritization, adaptive mutation
- FuzzGuard : DGF에 제공하기 전에 target site에 도달할 수 없는 input을 필터링하기위하여 딥러닝 사용
- *Leopard* : program matric을 사용하여 취약한 코드를 식별하여 이를 target site로 설정
- *ParmeSan* : sanitizer로 계측된 위치를 target site로 사용
- *CAFL* : target site, data condition을 사용하여 DGF의 특정 충돌 노출 능력 향상
## 6.3. Coverage-guided Grey-box fuzzing
- *AFL* coverage를 feedback으로 사용하는 classic CGF
- hybrid fuzzing은 CGF와 SE를 결합하여 CGF의 constraint solving능력을 강화
- *Steelix* : 경랑 정적 분석과 바이너리 계측을 사용하여 magic byte로 보호되는 path를 탐색하는데 도움이 되는 정보를 얻음
- *Angora* : SE 없이 constraint를 해결하기 위한 gradient descent based input serach를 사용
- *Greyone* : fuzzing-driven inference를 사용하여 data-flow를 사용하여 fuzzing을 도와줌
- *FairFuzz* : 드분 branch를 cover하는 input을 생성하는데 영향을 미치는 byte를 식별, 이외에도 seed scheduling process 개선, branch pentrating, detecting specific type of bug등을 사용함
  
*WindRanger*는 CGF에서 사용되는 일부 기술 (constraint solving technique) 등을 사용할 수 있음
# 7. Conclusion
- DGF의 맥락에서 DBB의 개념을 제안함.
- 이를 적용하여 *WindRanger* 제시
- 최신의 DGF에 비해 TTT (21%), TTE(44%) 향상
- 0-day vulnerability를 찾는데 도움을 주며 실용적인 유용성을 보여줌



# 8. study

## 8.1. DBB가 여러개 

## 8.2. effector map 에서 branch를 선정하는 기준

## 8.3. ExtractConstraintVar 함수
## 8.4. Simulated Annealing 
## 8.5. seed priorization
## 8.6. fuzzer test set 선정 기준

