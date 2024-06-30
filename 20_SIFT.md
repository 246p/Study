[Sound Input Filter Generation for Integer Overflow Errors](https://www.cs.toronto.edu/~fanl/papers/sift-popl14.pdf)

# 0. Abstract
- SIFT : memory allocation, block copy에서 INT OF를 막는 input filter를 생성
- 이전 기술과 다르게 sound함 (62895중 false postive 발생 X)

# 1. Introduction
## 1.1. Pervious Filter Generation Systems
### FSE
- 기존 방법은 error를 발생하는 input에서 시작하여 execution path를 찾음
- 이 path를 기반으로 FSE를 통해 vulnerability signature를 도출
- vulnerability signature : error와 동일한 path를 지나는 input의 조건식 (boolean condition)
- 이는 sound 하지 않음 > error로 가는 다른 path가 존재할 수 있기 때문
### weakest precondition analysis
- [기존방법](https://ieeexplore.ieee.org/document/4271657)의 두가지 문제점 (unsound)
1. loop unroling을 사용함 > 가능한 execution path의 subset만 분석
2. aliased value를 고려하지 않음
- 이렇게 생성된 조건은 execution path constraint와 통합되기 때문에 프로그램이 거를 수 있는 input을 filtering 하지 않음 > 많은 연산을 수행함
## 1.2. SIFT
- 
## 1.3. No Execution Path Constraints
### 1.3.1. Sound and Efficient Analysis
### 1.3.2. Efficient Filters
### 1.3.3. Accurate Filters
## 1.4. Input Fields With Multiple Instantiations
## 1.5. Pointer Analysis and Precondition Generation
## 1.6. SIFT Usage Model
## 1.7. Experimental Results
## 1.8. Contributions
# 2. Example
# 3. Static Analysis
## 3.1. Core Language and Notation
## 3.2. Intraprocedural Analysis
## 3.3. Interprocedural Analysis
## 3.4. Extension to C Programs
## 3.5. Input Filter Generation
# 4. Soundness of the Static Analysis
## 4.1. Dynamic Semantics of the Core Language
## 4.2. Soundness of the Pointer Analysis
## 4.3. Abstract Semantics
## 4.4. Relationship of the Original and the Abstract Semantics
## 4.5. Evaluation of the Symbolic Condition
## 4.6. Soundness of the Analysis
# 5. Experimental Results
## 5.1. Methodology
## 5.2. Analysis and Filter Evaluation-
## 5.3. Filter Confirmation on Vulnerabilities
## 5.4. Discussion
# 6. Related Work

# 7. Conclusion
