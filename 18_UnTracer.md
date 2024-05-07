![Full-speed Fuzzing: Reducing Fuzzing Overhead through Coverage-guided Tracing](https://users.cs.utah.edu/~snagy/papers/19SP.pdf)

# 0. Abstract
# 1. Introduction
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
