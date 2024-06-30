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
- critical expression : malloc, block copy의 size
- SIFT는 critical expression에서 시작하여 constrol flow에 따라 backward propagate
- interprocedural, demand-driven, weakest precondition SA 사용
- demand-driven : 특정 부분에 대한 정보가 필요할때 (demand가 있을때) analysis를 수행
- 이 결과는 critical expresion의 value를 갖기 위해 평가할 수 있는 모든 expression의 symbolic condition
- 즉 input field 에서 수행할 수 있는 모든 연산을 포착
- input이 주어지면 input filter는 해당 input field에 대해 조건을 평가하여 OF를 유발하는 input 폐기
- 모든 path를 고려하기 때문에 sound
## 1.3. No Execution Path Constraints
- 기존과 다르게 condition을 확인하는 부분을 포함하지 않고 ciritical expression의 value를 변경하는 arithmetic expression만 사용
- 이러한 선택의 특징
1. Sound and Efficient Analysis
- execution path constraint를 무시함으로써 더 효율적이다.
- target site로 가는 다양한 execution path에서 나타나는 constrasint를 추적할 필요가 없기 때문
- 이러한 path의 수가 매우 많기 때문에 execution path constraint를 완벽하게 도출하는것은 불가능함 > 기존 방법이 unsound한 이유
2. Efficient Filters
- coditional statement를 무시하기 때문에 input filter의 overhead가 낮음
3. Accurate Filters
- 하지만 무시하는 conditional statement가 safety check를 수행할 수 있음, 이를 무시하면 안전한 input 또한 폐기될 수 있음
- 하지만 실험 결과 이를 무시한는 것이 큰 정확도 손실을 초례하지 않음
## 1.4. Input Fields With Multiple Instantiations
- input file은 동일한 input field의 여러 instance를 포함함
- SIFT는 propagated symbolic expression의 free variable이 참조하는 input field의 모든 instance를 추상화함
- 즉 variable과 동일한 input field와 다른 instance간 대응을 시키지 않음
- 동일한 input field를 참조하는 모든 변수는 서로 교환 할 수 있음 > 정확한 대응 관계를 결정할 수 없는 프로그램 또한 분석 가능
- 교환 가능성은 loop에서 variable을 renumbering 하여 loop invarient expression을 얻는 알고리즘을 가능하게함 [3.2](#32-intraprocedural-analysis)
## 1.5. Pointer Analysis and Precondition Generation
- SIFT는 potential aliasing relationship을 분석하여 pointer를 equivalence set으로 gouping
- pointer을 통해 값을 load할때 이를 분석하기 위해 load 값을 나타내는 새 변수를 생성함
- 각 변수가 참조하는 정확한 값을 정적으로 결정할 수 없는 프로그램을 분석할 수 있음
- 이 논문은 arbitary off-the-shelf alias or pointer analysis와 precondition generation algorithm을 통합하는 첫 논문
## 1.6. SIFT Usage Model
1. Module Identification : input을 처리하는 module을 식별, SIFT는 이러한 module을 분석하여 해당 module이 처리하는 input에 대한 input filter 생성
2. Input Statement Annotation : 각 input statement가 읽는 input field를 식별하기 위해 주석을 추가 
3. Critical Site Identification : module을 scan하여 ciritical site를 식별, ciritical expression의 INT OF를 유발하는 input을 filtering
4. Static Analysis : ciritical exprssion에 대해 demand-driven backward SA를 통해 symbolic condition 도출, input field의 함수로 critical expression의 값이 어떻게 계산되는지 지정
5. Input Parser Acquisition : input format에 대한 parser를 획득, input bitstream을 group화 하여 API를 통해 field를 사용할 수 있음
6. Filter Generation : filter는 input field를 읽고 symbolic condition을 사용하여 input filter 생성

## 1.7. Experimental Results
- SIFT를 통해 5개의 real-world application에 적용,56/58개의 filter 생성
- filter 생성 시간 : 1초 미만
- 62895 개의 input 중 false postive 발생 X
- filter의 overhead 최대 16ms
## 1.8. Contributions
### 1.8.1. SIFT
- SIFT는 Module을 스캔하여 memory allocation, block copy site를 찾음
- symbolic condition을 도출하여 이에 따라 INT OF를 유발하는 input을 거르는 input filter를 자동으로 생성
- 기존과 다르게 모든 execution path를 고려하기 때문에 sound함
- execution path constraint를 무시하기 때문에 효율적임 (기존 수만번 -> SIFT 수십번)
### 1.8.2. Sound and Efficient Static Analysis
- input field
### 1.8.3. Input Fields With Multiple Instantiations
### 1.8.4. Pointer Analysis and Precondition Generation
### 1.8.5. Experimental Results
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
