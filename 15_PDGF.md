![Predecessor-aware Directed Greybox Fuzzing](https://csdl-downloads.ieeecomputer.org/proceedings/sp/2024/3130/00/313000a040.pdf?Expires=1714452425&Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9jc2RsLWRvd25sb2Fkcy5pZWVlY29tcHV0ZXIub3JnL3Byb2NlZWRpbmdzL3NwLzIwMjQvMzEzMC8wMC8zMTMwMDBhMDQwLnBkZiIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTcxNDQ1MjQyNX19fV19&Signature=CL4CF5f2c~oBEB1o6ZM6kOGEDFJ0WxSutK~Oadii33IGRHRN2uqaT6F0G14EP7bhihfOb0j9ermE-gCInPWbO-CJ~uGRbrIToPQ-x55NNQMJ8n785e1yh2EEJV8wG~d8OihQNbWcPSCIpV-dgcWad8wJDIkYKkgdHG3TkDxaOhdCa5v2zU-szAe2pTjzpuKJQLGH9iXDJKRuY3HCraCliYMF6G-RAMjibJelfeq67bhpwICiUYGcxdAzP~deJifgPZYoE-Fvaykeo74e-VM9VfDeEf5qUzrOvxYIXSpc6BU~qZq4D3hp6NPnvRqzTnHGaVMjHThKXDBzCkKrS4YWYg__&Key-Pair-Id=K12PMWTCQBDMDT)

# 0. abstract

# 1. Introduction

# 2. Background

## 2.1. Directed greybox fuzzing

## 2.2. Problem and challenges

### 2.2.1. Heavyweight

### 2.2.2. Incomplete

# 3. Predecessor-aware DGF

## 3.1. Framework

## 3.2. Predecessor identification

## 3.3. Predecessor-driven coverage tracing

## 3.4. Fuzzing based on regional maturity

### 3.4.1. Seed selection

### 3.4.2. Power scheduling

### 3.4.3. Seed mutation

# 4. Implementation

## 4.1. Initialization at compile-time

## 4.2. Managing two binaries

# 5. Evaluation

## 5.1. Evaluation setup

### 5.1.1. Benchmarks

### 5.1.2. Baselines

### 5.1.3. Settings

## 5.2. Bug reproducing capability (RQ1)

## 5.3. Path diversity (RQ2)

## 5.4. Understanding performance boost (RQ3)

## 5.5. Component investigation (RQ4)

## 5.6. Bug discovery (RQ5)

# 6. Discussion

## 6.1. Hash collision

## 6.2. Identify venerable path

## 6.3. Hybrid DGF

# 7. Related Work

## 7.1. Distance-based DGF

## 7.2. Non-distance-based DGF

# 8. Conclusion
