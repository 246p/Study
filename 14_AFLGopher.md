[Guiding Directed Fuzzing with Feasibility](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=10190644&tag=1)
# 0. Abstract
- 기존 distance calculation은 feasibility-unware
- 예로 if문의 두 분기가 같은 feasibility를 가진다고 가정 > DGF의 편향
- fesibility-aware DGF *AFLGopher* 제시
-
# 1. Introduction
# 2. Motivation
# 3. Approach
## 3.1. Overview
## 3.2. Branch Statement Analysis
### 3.2.1. Branch Statement Extraction
### 3.2.2. Branch Statement Grouping
## 3.3. Tracer
## 3.4. Feasibility Prediction
### 3.4.1. Branch Statement Feasibility Prediction
### 3.4.2. Indirect-call Target Feasibility Prediction
### 3.4.3. Distance Calculation
#### BB level
#### Function level
#### Distance table
## 3.5. Fuzzer Updating
#### Weights on CFG
#### Weights on CG
### 3.5.1. Error Monitor

# 6. Conclusion
