# 0. Abstract
# 1. Introduction
# 2. Background
## 2.1. Guided Fuzzing
## 2.2. Directed Fuzzing
# 3. PITFALLS OF DISTANCE MINIMIZATION
## 3.1. Experiment Setup
## 3.2. Consequence 1: Poor Performance
## 3.2. Consequence 2: Unconstrained Exploration
# 4. Overcoming the bottlenecks of directedness
# 5. Preemptive Termination
## 5.1. Tripwiring
### 5.1.1. Methodology
### 5.1.2. Indirect Transfers
# 6. Implementation : SIEVEFUZZ
## 6.1.  Architectural Overview
## 6.2. High-level Fuzzing Workflow
### 6.2.1. Initial Analysis (INIT)
### 6.2.2. Exploration (EXP)
### 6.2.3. Tripwired Fuzzing (FUZZ)
## 6.3. Maintaining Fast On-demand Analysis
## 6.4. Maintaining Fast SUT Execution
### 6.4.1. Preemptive Termination
### 6.4.2. Indirect Call Tracking
## 6.5. Maintaining Exploration Diversity
# 7. EVALUATION
## 7.0.1. Benchmarks
## 7.0.2. Experiment Procedure and Infrastructure
## 7.1. RQ1: Tripwiring’s Search Space Restriction
### 7.1.1. Results: Magnitude of Space Restriction
### 7.1.2. Results: Initialization Cost
### 7.1.3. Results: On-demand Analysis Cost
## 7.2. RQ2: Targeted Defect Discovery
### 7.2.1. Results: Tripwiring vs. Minimization-directed Fuzzing
### 7.2.2. Results: Tripwiring vs. Precondition-directed Fuzzing
### 7.2.3. Results: Tripwiring vs. Undirected Fuzzing
## 7.3. RQ3: Target Location Feasibility for Tripwiring
# 8. Discussion and Future Work
## 8.1. Refinements in Path Analysis
## 8.2. Path Prioritization
# 9. Related Work
## 9.1. Directed Fuzzing
## 9.2. Improving Fuzzing Performanc
# 10. Conclusion
