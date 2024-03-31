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

## 2.1. Challenge 1: Negative Feedback

## 2.2. Challenge 2: Misleading Distance Metrics

# 3. Overview

## 3.1. Selective Coverage Instrumentation

## 3.2. Semantic Relevance Scoring

# 4. Design
## 4.1. Program Slicing

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
