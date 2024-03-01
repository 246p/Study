[Binary-level Directed Fuzzing for Use-After-Free Vulnerabilities](https://www.usenix.org/system/files/raid20-nguyen.pdf)

# 0. Abstract
- DF는 추가 정보를 활용하여 bug reproductionm, patch testing, static analysis report verification 등에 사용된다. 
- UAF와 같은 어려운 취약점은 특히 binary level에서 다루어지지 않고 있다.
- 우리는 UAF에 집중하는 첫 binary-level DGF *UAFuzz*를 소개한다.
# 1. Background
## 1.1. Context
- *AFL*, *LIBFUZZER*와 같은 CGF는 code coverage 정보를 이용하여 input generation을 PUT의 새로운 부분으로 유도하고 많은 program state를 탐색하여 crash를 유발하려고 한다.
- DGF는 사전에 선택된 잠대적으로 취약산 target site에 대한 테스트를 수행한다.
- DGF의 Bug reproducing에 초점을 맞춘다.
- 이는 bug report 정보가 제공된 취약점의 Proof-of-Concept(PoC)을 기반으로 입력을 생성한다.
- PoC가 제공되더라도 개발자는 regression bug, incomplete fixe의 위혐으로 모든 코너 케이스를 고려해야한다.
- bug 가 발생할때 함수 호출 순서(bug stack trace)는 DF를 안내하는데 널리 사용된다.
- *ASan*, *VALGRIND*와 같은 도구에 PoC input으로 code를 실행하면 bug stack trace가 출력된다.
## 1.2. Problem
- GF는 UAF와 같은 복잡한 취약점을 찾기 어려움
- 우리는 UAF에 초점을 맞추고 있다.
## 1.3. Goals and chllenges
- UAF에 알맞은 DF을 설계함
- source code level instrumentation 없이 binary level에서 작동
- source code는 항상 사용할 수 없고 부분적으로 제3자 라이브러리에 의존할 수 있기 때문
  
다음과 같은 문제가 존재

### 1.3.1. Complexity
- UAF는 동일한 메모리 위치에서 3가지 event(할당, 해제, 사용)을 유발하는 입력을 생성해야함
- PUT의 여러 함수에 걸쳐 있음
- 시간적 공간적 제약은 만족시키기 어려움
### 1.3.2. Silence
- UAF는 SEGFAULT와 같은 관찰 가능한 효과가 없을때가 있음
- crash만 관찰하는 fuzzer는 testcase가 UAF를 윱라했다는것을 감지하지 못함
- runtime overhead로 인하여 *ASAN*, *VALGRIND*와 같은 profiling tool을 fuzzing에 사용할 수 없음

실제로 최신 DGF *AFLGo*, *HawkEye*는 이를 해결할 수 없음
- UAF의 특징인 temporality를 고려하지 못함
- metric의 sequenceness를 고려하지 못함
- UAF에 대한 사전 정보가 없기에 많은 seed를 다른 추가 검사를 위해 보냄
- source-based DGF는 instrumentation 단계가 오래 걸림
## 1.4. Proposal
- UAF에 특화된 binary level DGF인 UAFuzz
- DGF의 일반적 구성을 따르지만 UAF의 특성에 맞추어 component를 조정함

![table1]()

- distance matric : target function으로 이어지는 짧은 call chain을 선호, 이는 allocation, free 함수를 포함할 가능성이 높음
- seed selection : sequenceness-aware target similarity metric
- power schedule : cut-edge : 전체 target에 도달할 가능성이 높은 perfix path를 선호하는 방법

- bug triaging 단계에서 seed를 bug가 가능한 것과 아닌것으로 식별하며 프로파일링 도구에 대한 quirie를 많이 줄인다. 
## 1.5. Contributions
- UAF에 특화된 첫 DGF 설계 및 3가지 구성요소 (selection heuristic, power schedule, input metrics)에 대한 검토
- GF *AFL*, binary analysis tool platfor *BINSEC* 위에 구현함
- Benchmark test
- bug reproduction setting에서 다른 도구 들과 평가
- Patch testing에서 효과적임을 입증
# 2. Background
이 논문에서 사용되는 몇가지 개념 정리
## 2.1. Use After Free
### 2.1.1. execution
- execution은 input에 의행된 state의 순서
- execution trace는 가시적인 오류로 끝날때 crash가 됨
- fuzzer의 목표는 crash로 이어지는 input을 찾는것
### 2.1.2. UAF bugs
- 더이상 유효하지 않은 heap-allocation object에 대한 pointer dereferncing 할때 발생하는 버그
- Double-Free(DF) 는 특수한 case
### 2.1.3. UAF-triggering conditions
- allocation, free, use 를 순서대로 cover하는 input을 찾아야함
- 모두 같은 memory object를 참조
- 마지막 use는 일반적으로 즉시 crash를 일으키지 않음
  
``` c
char *buf = (char *)malloc(BUF_SIZE);
free(buf); // pointer buf becomes dangling
...
strncpy(buf, argv[1], BUF_SIZE-1); // UAF
```


## 2.2. Stack Traces and Bug Traces
- stack trace (function call list)를 추출 할 수 있음
- 이는 program state에 대한 순서를 제공하기 bug reproducing 관점에서 의미 있음
- UAF에 대한 carsh는 UAF가 발생한 후 오랜 시간이 지나서 발생하므로 표준 stack trace는 UAF를 재현하는데 도움이 되지 않음
- *ASan, VALGRIND*와 같은 동적으로 메모리 손상을 탐지하는 profiling tool의 경우 모든 메모리 관련 이벤트의 stack trace를 기록함
- 객체가 해제된 후 사용될 때를 감지하면 3개의 stack trace (alloc, free ,use)를 보고 받음 이를 (UAF) bug trace라고 부름
- 이러한 bug trace를 target이라고 함

![figure1]()


## 2.3. Directed Greybox Fuzzing
![algorithm1]()

# 3. Motivation

``` C
int *p, *p_alias;
char buf [10];
void bad_func (int *p) { free (p);} /* exit() is missing */
void func () {
    if ( buf [1] == ’F’)
        bad_func (p);
    else /* lots more code ... */
}
int main (int argc , char * argv []) {
    int f = open ( argv [1] , O_RDONLY );
    read (f , buf , 10) ;
    p = malloc (sizeof(int));
    if ( buf [0] == ’A’){
        p_alias = malloc (sizeof(int));
        p = p_alias ;
    }
    func() ;
    if ( buf [2] == ’U’)
        *p = 1;
    return 0;
}
```

12 : 할당, 10 : 헤제, 19 : 역참조를 통하여 UAF 발생
## 3.1. Bug-triggering conditions
- input의 첫 3byte가 "AFU"일때 발생
- alloc, free, use의 이벤트를 순서대로 커버해야 함
- crash가 발생하지 않으므로 sanitization 없이 오류를 탐지하지 못함
## 3.2. Coverage-based Graybox Fuzzing
CGF에서 "AFU"의 input을 생성할 확률은 낮기 때문에 개별 event가 쉽게 트리거 되더라도 UAF event의 순서를 추적하는데 효과적이지 않다.
## 3.3. Directed Greybox Fuzzing
- *ASan*에 의해 주어진 bug trace (14-alloc, 17,6,3-free,19-use)가 주어지면 DGF는 fuzzzer가 원하지 않는 path를 탐색하는것을 방지
- DGF는 특정한 순서로 위치에 도달하려고 시도하는것이 아닌 target site를 많이 cover하는 seed를 선호
- target (A,F,U)에 대해 A->F->U와 U->A->F를 구분하지 않음
- target site를 탐색할때 순서의 부재는 UAF bug 감지를 어렵개 한다.

## 3.4. glimpse
- 우리는 seed selection heuristic을 수정하는데 의존한다.
- execution trace가 corver하는 target의 수를 늘린다.
- target ordering-aware seed metric
- *AFL*에서 "AAAA"와 "AFAA"는 code coverage를 증가시키지 않기때문에 버려짐
-  하지만 target wsimilarity metric score에 최대값을 가지기 때문에 *UAFuszz*에서 선택됨
-  *AFAA*-> *AFUA*를 생성함

## 3.5. Evaluation
- *AFLGo*는 2시간 내에 버그 탐지 실패
- *UAFuzz*는 20분만에 탐지 성공
- *VALGRIND에 보내는 입력 또한 낮음 

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