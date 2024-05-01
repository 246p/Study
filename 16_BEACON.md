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
- BEACON : 적은 overhead로 infeasible path를 pruning
- lightweight SA를 이용하여 infeasible 하게 만드는 variable에 대한 근사치 계산 가능

![figure2](./image/16_figure2.png)

> Contribution
- cheqp cost SA를 이용하여 target에 도달하기 위한 condition을 계산하고 이를 이용하여 infeasible program state를 filtering
- runtime overhead가 낮고 많은 infeasible path를 pruning 하는 DGF 제시

# 2. Background
## 2.1. Directed Grey-Box Fuzzing
- DGF의 목표 : 프로그램의 특정 부분을 작은 runtime overhead로 testing 하는것
- challenge
1. 어떤 대상을 테스트 할지 지정
2. fuzzer를 target code에 빠르게 도달하게 하는방법
### 2.1.1. Specifying the Targets
- 수동으로 target code를 지정할 수 있음 (patch가 이루어진 곳)
- 자동으로 지정 하는 방법(*Semfuzz* : 자연어 처리를 활용한 bug report 분석, *ParmeSan* : sanitizer에 의해 도입된 지점을 labeling)
### 2.1.2 Reaching the Targets
- AFLGo 
- Hawkeye
- FuzzGuard
- Savior
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
