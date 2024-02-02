[Angora: Efficient Fuzzing by Principled Search](https://web.cs.ucdavis.edu/~hchen/paper/chen2018angora.pdf)

# 0. Abstract
fuzzing은 소프트웨어 버그를 찾기 위해 널리 사용되고 있다. 그러나 최신의 fuzzer의 성능은 기대에 못미치기도 한다.

- symbolic execution 기반 : high quality input , 속도가 느림
- random muatation 기반 : low quality input, 속도가 빠름

이 논문에서는 새로운 mutation-based fuzzer인 Angora를 제시한다.

Angora의 목표는 symbolic execution 없이 path constraint를 해결함으로 branch coverage를 증가시키는 것이다. 다음과 같은 핵심 기술을 도입하였다.

- scalable byte-level taint tracking
- context-sensetive branch count
- gradient descent based search
- input legnth exploration

LAVA-M data set에서 거의 모든 injected bug를 찾았고 다른 모든 fuzzer보다 더 많은 bug를 찾았다.

*who* 프로그램에서는 두번째로 좋은 fuzzer보다 8배 많은 버그를 찾았다. 또한 LAVA 제작자가 injected 하였지만 trigger하지 못한 103개의 버그를 찾아내었다. 

또한 8개의 널리 사용되는 open source program에서 Angora를 테스트하여 다음과 같은 버그를 발견하였다

|program name| number of bug|
|:---:|:---:|
|file|6|
|jhead|52|
|nm|29|
|objdump|40|
|size|48|

# 1. Introduction

# 2. Background: American Fuzzy Lop (AFL)

## 2.1. Branch coverage

## 2.2. Mutation strategies

# 3. Design

## 3.1. Overview

## 3.2. Context-sensitive branch count

## 3.3. Byte-level taint tracking

## 3.4. Search algorithm based on gradient descent

## 3.5. Shape and type inference

## 3.6. Input length exploration

# 4. Implementation

## 4.1. Instrumentation

## 4.2. Fuzzer

# 5. Evaluation

## 5.1. Compare Angora with other fuzzers

## 5.2. Evaluate Angora on unmodified real world programs

## 5.3. Context-sensitive branch count

### Performance

### Hash collision

## 5.4. Search based on gradient descent

## 5.5. Input length exploration

## 5.6. Execution speed

# 6. Related work

## 6.1. Prioritize seed inputs

## 6.2. Taint-based fuzzing

## 6.3. Symbolic-assisted fuzzing

# 7. Conclusion

# 8. Acknowledgment
