[Full-speed Fuzzing: Reducing Fuzzing Overhead through Coverage-guided Tracing](https://users.cs.utah.edu/~snagy/papers/19SP.pdf)

# 0. Abstract
- CGF의 3가지 구성 요소
1. test case generation
2. code coverage tracing
3. crash triage
- code coverage tracing이 가장 큰 overhead > static or dynamic binary intrumentation을 통한 모든 test case의 code coverage 추적 > 대다수의 coverage 정보가 code coverage를 증가시키지 않아 버려짐
- 이를 해결하기 위하여 coverage guided tracing 도입
1. 생성된 test case중 소수만이 coverage를 증가시킴
2. coverage를 증가하는 test case는 시간이 지남에 따라 드물게 발생
- target bianry file에 현재 frontier of coverage를 incoding하여 testcase가 새로운 coverage를 생성할때 tracing 없이 self-report > filtering
- coverage를 증가시키지 않는 testcase를 처리하는 시간을 줄임
- UnTracer : Static binary intrumentation tool *Dyninst*를 기반으로 구현
- testcase를 더 많이 실행할 수 있음
# 1. Introduction
- code coverage는 3가지 형태 > BB, BB edge, BB path
- White box : compile time에 intrumentation을 통한 coverage 측정
- Black box : 동적으로 삽입된 intrumentation or binary rewriting, or hardware support
- code coverage를 증가 시키는 것은 1/10000 이기에 모든 test case의 coverage를 추적하는것은 낭비
- UnTracer : CGF에서 overhead를 줄이는 것이 목표
- coverage guided tracing은 code coverage 를 증가시키는 test case에 대해서만 보고하도록 함 > 이러한 수정된 binary 를  *Interst Oracle*
- 
# 2. Background
## 2.1. An Overview of Fuzzing
## 2.2. Coverage-Guided Fuzzing
## 2.3. Coverage Tracing Performance
## 2.4. Focus of this Paper
# 3. Impact of Discarded Test Cases
## 3.1. Experimental Setup
## 3.2. Results
# 4. Coverage-guided Tracing
## 4.1. Overview
## 4.2. The Interst Oracle
## 4.3. Tracing
## 4.4. Unmodifying
## 4.5. Theoretical Performance Impact
# 5. Implementation : UnTracer
## 5.1. UnTracer Overview
## 5.2. Forkwerver Instrumentation
## 5.3. Interst Oracle Binary
## 5.4. Tracer binary
## 5.5. Unmodifying th Oracle
# 6. Tracing-only Evaluation
## 6.1. Evaluation Overview
## 6.2. Experiment Infrastructure
## 6.3. Benchmarks
## 6.4. Timeouts
## 6.5. Untracer vs Coverage-agnostic Tracing
## 6.6. Dissecting UnTracer's Overhead
### 6.6.1. Tracing
### 6.6.2. Forkserver restarting
## 6.7. Overhead vs Rate of Coverage-increasing test cases
# 7. Hybrid Fuzzing Evaluation
### 7.0.1. Trimming and calibration
### 7.0.2. Saving timouts
## 7.1. Evaluation Overview
## 7.2. Performance of UnTracer-based Hybrid Fuzzing
# 8. Discussion
## 8.1. UnTracer and Intel Processor Trace
## 8.2. Incorporating Edge Coverage Tracking
## 8.3. Comprehensive Black-Box Binary Support
# 9. Related Work
## 9.1. Improving Test Case Generation
## 9.2. System Scalability
# 10. Conclusion
