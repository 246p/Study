[Constraint-guided Directed Greybox Fuzzing](https://www.usenix.org/system/files/sec21fall-lee-gwangmu.pdf)

# 0. Abstract
- DGF는 orderd target site, data condition을 고려하지 않음
- target site에 도달하는 것이 아닌 constraint를 만족하는 것을 목표로 하는 constraint-guided DGF를 제시
- target site와 dat condition의 조합으로 constraint를 정의하고 해당 constraint를 만족하도록 seed를 유도함
- 7가지 crash dump와 4가지 patch changelog로 constraint 생성
- constraint가 명확해질수록 PoC generation 측면에서 우수한 성능을 보여줌
# 1. Introduction
- 현재 DGF의 문제
1. DGF는 target site 들이 서로 독립적이라고 가정, target site의 order dependency를 고려하지 않음
2. target crash에 필요한 data condition을 고려하지 않음
- 이러한 문제를 해결하는 CDGF 제안
- target site에 도달하는것이 목표가 아닌 constraint를 만족시키는것을 목표로함
- 순서에 따라 그 constraint를 더 잘 만족시키는 seed를 우선
- constraint는 하나 이상일 수 있으며 이때 지정된 순서대로 constraint를 만족해야함
# 2. Background and Motivation

## 2.1. Directed Greybox Fuzzing

## 2.2. Usage Example

### 2.2.1. Static Analyzer Verification

### 2.2.2. Crash Reproduction

### 2.2.3. PoC Generation

## 2.3. Limitation

### 2.3.1. Independent Target Sites

### 2.3.2. No Data Condition

## 2.4. Requirements

# 3. Constraint-guided DGF

## 3.1. Overview

## 3.2. Example

### 3.2.1. Ordered Target Sites

### 3.2.2. Data Conditions

# 4. Constraints

## 4.1. Definition

### 4.1.1. Variable capturing

### 4.1.2. Data condition

### 4.1.3. Orderedness

## 4.2. Distance of Constraints

### 4.2.1. Target Site Distance

### 4.2.2. Data Condition Distance

### 4.2.3. Constraint Distance

### 4.2.4. Total Distance

# 5. Constraint Generation

## 5.1. Crash Dump

### 5.1.1. Multiple Target Sites (nT)

### 5.1.2. Two Target Sites with Data Conditions (2T+D)

### 5.1.3. One Target Site with Data Conditions (1T+D)

## 5.2. Patch Changelog

# 6. Implementation

## 6.1. System Overview

## 6.2. CAFL Compiler

## 6.3. CAFL Runtime

## 6.4. CAFL Fuzzer

# 7. Evaluation

## 7.1. Microbenchmark: LAVA-1

## 7.2. Crash Reproduction

## 7.3. PoC Generation

# 8. Discussion

## 8.1. Use-cases with manually written constraints

## 8.2. Bugs that require three or more constraints

## 8.3. Ineffective scenarios

## 8.4. Bugs that require further research

## 8.5. Issue on distance of pointer conditions

# 9. Related Work

## 9.1. Directed greybox fuzzing

## 9.2. Static analysis-assisted directed fuzzing

## 9.3. Targeted analysis with symbolic execution

## 9.4. ML-based directed fuzzing

## 9.5. Alternative distance metrics

## 9.6. Domain-specific fuzzing

## 9.7. PoC generation

# 10. Conclusion
