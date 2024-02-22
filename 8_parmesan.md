[ParmeSan: Sanitizer-guided Greybox Fuzzing](https://download.vusec.net/papers/parmesan_sec20.pdf)

# 0. Abstract
# 1. Introduction
# 2. Background
## 2.1. Fuzzing strategy
## 2.2. Directed fuzzing
## 2.3. Target selection with sanitizers
## 2.4. CFG construction
# 3. Overview
## 3.1. Target acquisition
## 3.2. Dynamic CFG
## 3.3. Fuzzer
# 4. Target acquisition
## 4.1. Finding instrumented points

## 4.2. Sanitizer effectiveness
## 4.3. Profile-guided pruning
## 4.4. Complexity-based pruning

# 5. Dynamic CFG
## 5.1. CFG construction
## 5.2. Distance metric
## 5.3. Augmenting CFG with DFA
# 6. Sanitizer-guided fuzzer
## 6.1. DFA for fuzzing
## 6.2. Input prioritization
## 6.3. Efficient bug detection
## 6.4. End-to-end workflow
# 7. Implementation
## 7.1. Limitations
# 8. Evaluation
## 8.1. ParmeSan vs. directed fuzzers
## 8.2. Coverage-guided fuzzers
## 8.2. Sanitizer impact
## 8.4. New bugs
# 9. Related Work
# 10. Conclusion