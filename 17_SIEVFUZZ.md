[One Fuzz Doesn’t Fit All: Optimizing Directed Fuzzing via Target-tailored Program State](https://dl.acm.org/doi/pdf/10.1145/3564625.3564643)


[source code](https://github.com/HexHive/SieveFuzz)

# 0. Abstract
- GF는 가까운 test case를 분리하고 이를 확률적으로 번형하는 distance minimization 전략을 사용함
- target에 도달하는지 여부와 관계없이 모든 test case에 대해서 적용됨 > target에 도달할 수 없는 path 에서 정체
- target에 도달 가능한 path를 우선시해야함
- 비효율적인 탐색 병목 현상을 극복하기 위하여 tripwiring 도입 > target site에 도달하지 않을 path를 사전에 차단하고 종료
- target site에 도달하지 않을 fuzzing을 사ㅓㄴ에 종료
- search를 target rechable path set으로 제한하여 search noise를 줄임
# 1. Introduction
- DGF에서 모든 testcase 에 대한 distance 측정으로 인하여 높은 instrumentation overhead
- tripwiring : distance minimization을 벗어나고 target unrechable path를 filtering

> contribution
- tripwiring : target site에 reach하는데 필수적인 program search space만 제한하는 target-tailored DF를 위한 ligthweight technique
- tripwiring이 distance minimization보다 최적화된 방법론임을 보여줌
- SieveFuzz 설계
# 2. Background
## 2.1. Guided Fuzzing
- guided fuzzing은 serach를 제어하는 feedback loop를 사용
- CGF, resource consumption, memory allocations, program state등 policy를 사용
## 2.2. Directed Fuzzing
- patch testing과 같은 search target을 위하여 DF 개념을 도입
- Fuzzing을 특정 target site로 "direction"
- 대부분 최신 DF는  distance minimazation을 선택함 
# 3. Pitfalls of Distance Minimization
## 3.1. Experiment Setup
## 3.2. Consequence 1: Poor Performance
## 3.2. Consequence 2: Unconstrained Exploration
# 4. Overcoming the bottlenecks of directedness
# 5. Preemptive Termination
## 5.1. Tripwiring
### 5.1.1. Methodology
### 5.1.2. Indirect Transfers
# 6. Implementation : SIEVEFUZZ
## 6.1.  Architectural Overview
## 6.2. High-level Fuzzing Workflow
### 6.2.1. Initial Analysis (INIT)
### 6.2.2. Exploration (EXP)
### 6.2.3. Tripwired Fuzzing (FUZZ)
## 6.3. Maintaining Fast On-demand Analysis
## 6.4. Maintaining Fast SUT Execution
### 6.4.1. Preemptive Termination
### 6.4.2. Indirect Call Tracking
## 6.5. Maintaining Exploration Diversity
# 7. EVALUATION
## 7.0.1. Benchmarks
## 7.0.2. Experiment Procedure and Infrastructure
## 7.1. RQ1: Tripwiring’s Search Space Restriction
### 7.1.1. Results: Magnitude of Space Restriction
### 7.1.2. Results: Initialization Cost
### 7.1.3. Results: On-demand Analysis Cost
## 7.2. RQ2: Targeted Defect Discovery
### 7.2.1. Results: Tripwiring vs. Minimization-directed Fuzzing
### 7.2.2. Results: Tripwiring vs. Precondition-directed Fuzzing
### 7.2.3. Results: Tripwiring vs. Undirected Fuzzing
## 7.3. RQ3: Target Location Feasibility for Tripwiring
# 8. Discussion and Future Work
## 8.1. Refinements in Path Analysis
## 8.2. Path Prioritization
# 9. Related Work
## 9.1. Directed Fuzzing
## 9.2. Improving Fuzzing Performanc
# 10. Conclusion
