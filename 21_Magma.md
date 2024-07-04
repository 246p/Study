[Magma: A Ground-Truth Fuzzing Benchmark](https://dl.acm.org/doi/pdf/10.1145/3428334)

# 0. Abstract
# 1. Introduction
# 2. Background and Motivation
## 2.1. Fuzz testing (fuzzing)
## 2.2. The Current State of Fuzzer Evalutaion
### 2.2.1. Existing Fuzzer Benchmarks
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
