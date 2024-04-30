[BEACON : Directed Grey-Box Fuzzing with Provable Path Pruning](https://qingkaishi.github.io/public_pdfs/SP22.pdf)

# 0. Abstract
- target code에 unrechable path를 symbolically or concretely execution 하여 resource 낭비
- target에 도달하는데 필요한 abstracted precondition을 lightweight static analysis로 계산
- 관련 없는 path를 제거하고 제거된 path가 target과 관련 없음을 보장
# 1. Introduction
- DGF의 효율성을 높이기 위해 unrechable execution path를 빠르게 중단하는것이 중요
- 이를 infeasible-path-explosion problem이라고 함
- DWF : SE를 이용하여 path constraint를 해결하여 rechability를 결정함 > scale의 문제가 있음
- DGF : unrechable path를 거부하는데 관심이 없음
- BEACON :
# 2. Background

## 2.1. Directed Grey-Box Fuzzing

### 2.1.1. Specifying the Targets

### 2.1.2 Reaching the Targets

## 2.2. Problem and Challenges

# 3. Beacon in a Nutshell

## 3.1. Backward Interval Analysis

## 3.2. Selective Instrumentation

# 4. Methodology

## 4.1. Preliminary

### 4.1.1. Language

### 4.1.2. Precondition Inference

## 4.2. Backward Interval Analysis

## 4.3. Optimizations for Maintaining Precision

### 4.3.1. Relationship Preservation

### 4.3.2. Bounded Disjunctions

## 4.4. Precondition Instrumentation

# 5. Evalutation

### 5.0.1. Baselines

### 5.0.2. Benchmarks

### 5.0.3. Configurations

## 5.1. Compared to the State of the Art

## 5.2. Impacts of Path Slicing & Precondition Checking

## 5.3. Impacts of Relation Preservation & Bounded Disjunction

## 5.4. Instrumentation Overhead

## 5.5. Case Study

## 5.6. Discussion

# 6. Related Work

## 6.1. Directed White-box Fuzzing

## 6.2. Coverage-guided Fuzzing

# 7. Conclusion
