[Magma: A Ground-Truth Fuzzing Benchmark](https://dl.acm.org/doi/pdf/10.1145/3428334)

# 0. Abstract
- fuzzer의 성능을 평가하고 비교하는 것은 metric과 benchmark의 부족으로 인해 어렵다...
- crash count : deduplication technique의 불완전성으로 인해 부정확함
- 특정 target set이 없기에 평가가 일관되지 않음
- Magma : Ground truth fuzzing benchmark
# 1. Introduction
- fuzzing : sound, incomplete
- 일반적으로 crash count, bug count, code coverage profile을 통해 형가됨
- 각 metric의 결함
1. crash count : 실제 bug의 수를 over approximate, deduplication technique의 불완전성
2. bug count : crash의 근본 원인을 식별하는 것이 더 바람직함, 하지만 ground-truth bug count를 얻기 위해선 수작업 분류가 필요하며 도메인 전문 지식과 경험 이 필요
3. code coverage profile : coverage로 제거된 충돌과 ground-truth bug간의 약한 상관관계, 높은 coverage가 반드시 더 좋진 않음
- 실제 프로그램에서 발생한 버그를 기반으로 7개 프로그램에 118개의 버그를 다시 삽입함
- 버그에 도달한것과 트리거 하는것을 구분함
- contributions
1. fuzzer benchmark를 위한 bug-centric performance metric을 제시
2. 기존 fuzzer benchmark의 정량적 비교
3. 실제 프로그램, 버그를 기반으로 한 ground-truth fuzzing benchmark
4. 7개의 fuzzer에 대한 Magama의 평가
# 2. Background and Motivation
## 2.1. Fuzz testing (fuzzing)
- fuzzer는 input format, program strucutre에 따라 input을 생성
- grammer based fuzzer (Superion, Peachfuzz, Quickfuzz) : input format 을 활용한 input 생성
- mutation based fuzzer (AFL, Angora, MemFuzz) : muatation을 통해 input 생성
-  fuzzing은 확률적으로 bug를 찾음, 이러한 특성으로 fuzzer를 평가하고 비교하는 것이 어려움
## 2.2. The Current State of Fuzzer Evalutaion
- 지금 까지 fuzzer를 평가한는 것은 임시적이고 무작위적임
- 다음과 같은 평가되어야 함
1. performance metric : crash count, bug count, coverage profiling
2. target : 다양하고 현실적이어야함
3. seed selection : initial input set이 fuzzer간 일관되어야함
4. Trial duration (timeout) : timeout이 일관되어야함
5. number of trial : fuzzer가 확률적이기에 통계적으로 신뢰 있는 비교를 위해 많은 반복이 필요
### 2.2.1. Existing Fuzzer Benchmarks
-
### 2.2.2. Crashes as a Performance Metric
# 3. Desired Benchmark Properties
## 3.1. Diversity (P1)
## 3.2. Verifiability (P2)
## 3.3. Usability (P3)
# 4. Magma : Approach
## 4.1. Target Selection
## 4.2. Bug Selection and Insertion
## 4.3. Performance Metrics
## 4.4. Runtime Monitoring
# 5. Design and Implementation Decisions
## 5.1. Forward-porting
### 5.1.1. Forward-Porting vs. Back-Porting
### 5.1.2. Manual Forward-Porting
## 5.2. Weired State
## 5.3. A Static Benchmark
## 5.4. Leaky Oracles
## 5.5. Proofs of Vulnerability
## 5.6. Unknown Bugs
## 5.7. Fuzzer Compatibility
# 6. Evalutaion
## 6.1. Methodology
## 6.2. Time to Bug
## 6.3. Experiimental Results
### 6.3.1. Bug Count and Statistical Significance
### 6.3.2. Time to Bug
### 6.3.3. Achilles’ Heel of Mutational Fuzzing
### 6.3.4. Magic Value Identification
### 6.3.5. Semantic Bug Detection
### 6.3.6. Comparison to LAVA-M
## 6.4. Discussion
### 6.4.1. Ground Truth and Confidence
### 6.4.2. Beyond Crashes
### 6.4.3 Magma as a Lasting Benchmark
# 7. Conclusions
