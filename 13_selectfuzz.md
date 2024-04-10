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
- 도달하지만 inrelevant code를 실행하여 에너지 낭비

# 3. Problem statement
## 3.1. Relevant Code
- relevant code : target에 도달(control flow condition), vulnerability에 사용 (data flow condition) 하는 코드
## 3.2. Limitations of Existing Approaches
## 3.3. Research Goals and Challenges
# 4. SelectFuzz
## 4.1. Block Distance
## 4.2. Selective Path Exploration
### 4.2.1. Relevant Code Identification
### 4.2.2.  Input Prioritization and Power Scheduling

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
