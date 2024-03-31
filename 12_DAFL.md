[DAFL: Directed Grey-box Fuzzing Guided by Data Dependency](https://kihongheo.kaist.ac.kr/publications/sec23.pdf)

# 0. Abstract
- DGF이 복잡한 프로그램에 있어서 2가지 확장성 문제
1. coverage feedback이 target에 도달하는데 유의미한 정보를 제공하지 못함
2. seed distance는 복잡한 control flow를 갖는 프로그램에서 잘 작동하지 않음
- DAFL은 target site와 관련된 코드를 식별하고 해당 부분으로 부터만 coverage feedback을 받
- data flow semantics를 고려하여 정밀한 seed distance 계산
# 1. Introduction
- 현재 DGF는 다음 두가지 핵심 매커니즘에 기반
1. 새로운 execution path를 cover하는 testcase를 선호
2. CFG내의 실행된 노드와 target 노드 사이의 거리에 따라 각 테스트 케이스의 우선순위를 지정함
> 복잡한 control flow를 갖는 프로그램에서 좋지 않음 
## 1.1. Challenge 1
- code coverage는 DGF에 부정적인 영향을 줄 수 있음
- 더 많은 coverage를 달성 할 수 있다면 target과 관련 없는 path로 유도할 수 있기 때문
- target program이 클 경우 이는 더 심화됨
- Beacon은 target에 도달하기 위한 weakest precondition을 계산하고 이를 이용해 실행을 조기에 중단시킴
- 실제 프로그램에서 정확한 weakest precondition을 계산하는것이 불가능 > [복잡한 loop는 이를 무력화시킴](#21-challenge-1-negative-feedback)

## 1.2. Challenge 2
- 현재의 distance mechanism은 복잡한 control flow를 갖는 프로그램에서 작 작동하지 않음
- CFG에서 실행된 모든 node를 고려하여 seed distance(priority score)를 계산함
- 프로그램이 큼 > path가 길어짐 > 무관한 node를 많이 포함할 가능성이 있음
- WindRanger는 DBB개념을 도입하여 이러한 문제를 처음으로 다룸 > [loop 내부에서는 DBB가 좋지 않음](#22-challenge-2-misleading-distance-metrics)
## 1.3. Our Solution
- *selective coverage instrumentation* : target과 관련된 부분의 code coverage를 수집하여 부정적인 feedback 줄임
- *semantic relevance scoring* : seed distance를 더 직관적으로 계산 > Def-Use-Graph (DUG)를 사용하여 loop 제거

## 1.4. Contribution
- selective coverage instrumetation
- semantic relevance scoreing
- DAFL 설계 및 공개
# 2. Motivation
`CVE-2017-7578`

![figure1](./image/12_figure1.png)

- `type == 42`일 때 버그가 발생
## 2.1. Challenge 1: Negative Feedback
![figure2](./image/12_figure2.png)
- 위의 예시에서 blockParse는 80개 이상의 함수를 호출할 수 있지만 `parseSWF_DEFINEMORPHSHAPE`만이 crash를 유발
- 기존의 DGF는 새롭지만 무관한 testcase에 많은 에너지를 할당함
- Beacon은 target에 도달할 수 없는 경롤르 제거하지만 복잡한 프로그램을 처리할 수 없음
## 2.2. Challenge 2: Misleading Distance Metrics
- 현대 DGF는 CGF의 syntatic distance에 기반 > syntatic distance는 semantic적인 측면을 반영 불가

![figure3a](./image/12_figure3.png)

> sa : 잘못된 format으로 즉시 종료, sb : 형식이 올바르고 target버그와 더 관련이 깊음

- 3a : AFLGo : sa = 34.3 < sb = 36.75
- 3b : WindRanger : sa = 34 < sb = 45

- 따라서 loop와 같은 복잡한 control flow가 있는 경우 WindRanger의 DBB도 잘못 될 수 있음 > 우리는 semantic relevance scoring을 이용하여 distance feedback mechanism을 도입
# 3. Overview
![figure4](./image/12_figure4.png)

- DAFL은 static analysis, fuzzing 두 단계로 구성됨
- SA에서 target site를 가진 프로그램을 입력으로 받아 data-dependent가 있는 모든 문장을 식별하기 위하여 inter-procedural static analysis를 수행하여 Def-Use Graph와 target point와 관련된 function set을 반환함
- [selective coverage instrumentation](#31-selective-coverage-instrumentation)을 통하여 관련된 함수를 instrument
- 이를 이용하여 fuzzing중 선택적인 coverage feedback을 받을 수 있음
- fuzzing 단계에서 [Semantic Relevance Scoring](#32-semantic-relevance-scoring)이라는 새로운 scheduling algorithm을 사용하여 DUG에서 파생된 relevance score에 따라 seed의 우선순위를 지정



## 3.1. Selective Coverage Instrumentation

- DAFL은 target site와 관련된 일부분으로부터 coverage feedback을 받음
- DUG상의 target 위치에서 시작하여 프로그램을 역방향으로 탐색하여 모든 dependent statement를 수집
- 이러한 node set은 target site와 semantic하게 관련된 부분을 포함 하지만 data dependency는 control dependency를 놓칠 수 있음 > 예를들어 line 29는 target point에 도달하는데 중요한 정보이지만 data dependency의 일부가 아니다.
- 만약 모든 data, control dependency를 수집한다면 정밀도 손실이 발생할 수 있다.
- slice된 DUG의 모든 함수를 수집하고 모든 node에 대해서 coverage feedback을 받음

## 3.2. Semantic Relevance Scoring
- seed input을 syntactic distance가 아닌 semantic distance로 평가함
- seed input score을 DUG에서 실행된 node의 semantic relevance score의 합으로 정의함
- 사장 먼 node에 1을 할당 하고 target으로 부터 가까운 node에 최대 점수를 부여함

# 4. Design
![algorithm1](./image/12_algorithm1.png)
- DAFL은 P(program), t(target program point)를 입력으로 받아 crash test case set을 반환
1. Static analysis phase
- P의 data dependency를 분석하여 t에대한 P를 slicing함 (line 1)
- slice함수는 tuple(G,F) 반환 > G : t에 대한 DUG, F : G에 의해 cover된 함수의 집합
- slice된 함수 집합 (F)는 selective coverage instrumentation을 수행하여 intrument됨 프로그램 P'을 제공
2. Fuzzing phase
- seed pool, test case set 초기화
- seed pool에서 seed를 선택하고 semantic relevance score에 기반하여 에너지 할당 (몇번 mutation을 수행할지 지정)
- mutated test case에 대하여 selective coverage를 측정

# 4.1. Program Slicing
- 전통적인 program slicing은 주어진 point에 영향을 줄 수 있는 모든 source line으 집합을 계산 > 하지만 이는 불필요하게 큰 slice를 생성 > 예제
``` C
x = f();
y = g();
p = &y;
z = x + *p;
```

- z에 대해 static program slice를 계산 > 단순한 slicing algorithm은 line 1~3 반환 > 실제로는 line 3는 관련 없음
- thin slicing은 pointer dereference를 무시하여 이를 해결함

https://chat.openai.com/c/7f14c5f7-a30e-4411-b6e0-5505cf61ce4c
## 4.2. Seed Scheduling

### 4.2.1. Semantic Relevance Score
### 4.2.2. Seed Pool Management
### 4.2.3. Energy Assignment

## 4.3. Implementation

# 5. Evaluation
## 5.1. Evaluation Setup
## 5.2. Time-to-Exposure
## 5.3. Impact of Thin Slicing
## 5.4. Impact of Selective Coverage Instrumentation & Semantic Relevance Scoring

# 6. Discussion
# 7. Threats to Validity
# 8 .Related Work
# 9. Conclusion
