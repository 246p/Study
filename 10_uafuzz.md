[Binary-level Directed Fuzzing for Use-After-Free Vulnerabilities](https://www.usenix.org/system/files/raid20-nguyen.pdf)

# 0. Abstract
# 1. Background
## 1.1. Context
## 1.2. Problem
## 1.3. Goals and chllenges
## 1.4. Proposal
## 1.5. Contributions
# 2. Background
## 2.1. Use After Free
## 2.2. Stack Traces and Bug Traces
## 2.3. Directed Greybox Fuzzing
# 3. Motivation
## 3.1. Bug-triggering conditions
## 3.2. Coverage-based Graybox Fuzzing
## 3.3. Directed Greybox Fuzzing
## 3.4. Gilmpse at UAFuzz
## 3.5. Evaluation

# 4. The UAFuzz Approach
## 4.1. Bug Trace Flattening
## 4.2. Seed Selection based on Target Similarity
### 4.2.1. Seed Selection
### 4.2.2. Target Similarity Metrics
### 4.2.3. Combining Target Similarity Metrics
## 4.3. UAF-based Distance
### 4.3.1. Zoom: Background on Seed Distance
### 4.3.2. Our UAF-based Seed Distance
## 4.4. Power Schedule
### 4.4.1. Cut-edge Coverage Metric
### 4.4.2. Energy Assignment

## 4.5. Postprocess and Bug Triage

# 5. Implementation
# 6. Experimental Evaluation
## 6.1. Resarch Question
## 6.2. Evaluation Setup
## 6.3. UAF Bug-reproducing Ability (RQ1)
## 6.4. UAF Overhead (RQ2)
## 6.5. UAF Triage (RQ3)
## 6.6. Individual Contribution (RQ4)
## 6.7. Patch Testing and Zero-days
## 6.8. Threat to Validity
### 6.8.1. Implementation
### 6.8.2. Benchmark
### 6.8.3. Competitors

# 7. Related Work
## 7.1. DGF
## 7.2. CGF
## 7.3. UAF Detection
## 7.4. UAF Fuzzing Benchmark

# 8. Conclusion