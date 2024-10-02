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
![figure2]()

- UAF를 발생하는 control-flow (시간적) runtime(공간적)조건을 모두 충족하는 입력을 찾기
- UAF 특성을 적용해서 bug trace에 따라 target에 도달하는 잠재적 inputd 
1. seed metric
- distance metric : target까지 거리를 근사, 3가지 event를 순서대로 cover할 수 있는지 확인
- cut-edge metric : 중요한 node에서 올바른 결정을 내리는지 확인
- target similarity mmetric : execution trace가 runtime에 cover하는 target의 수
2. seed selection strategy
- runtime에 많은 target을 cover
- matric score 기반 power scheduler 

3. profiling tool
- matric을 이용한 버그 확인을 위하여 *VALGRIND*를 이용하여 가능성이 높은 PoC input을 사전에 식별하여 불필요한 검사 줄임


## 4.1. Bug Trace Flattening
- bug traces는 복잡하여 light-weight intrumentation에 적합하지 않음
- bug trace flattening : 통과한 BB의 sequence 추출
  

![figure3]()

1. 3개의 stak trace를 각 call tree의 path로 보고 stack trace를 병합하여 해당 tree를 재생성함, 여러 child를 가진 node는 UAF event 순서에 따라서 정렬함
2. preorder traversal을 이용하여 target site에 대한 sequence를 얻음
## 4.2. Seed Selection based on Target Similarity
우리의 seed selection은 다음 두가지에 기반함
1. DF가 target bug trace cover하는 bug를 찾기 위하여 target bug trace와 유사한 seed 우선
2. target similarity 는 ordering을 고려해야 한다.
### 4.2.1. Seed Selection

> Defenition 1 : max reaching input

![algorithm 2]()

- max-reaching input : `target bug trace T`와 가장 유사한 input
- 유사도는 `target similarity metric t(s,T)`에 의해 결정
- max-reaching input을 선택하고 mutation 해야 하지만 작은 확률 α(1%, AFL과 동일)로 code coverage를 위한 seed를 선택함
### 4.2.2. Target Similarity Metrics
![figure4]()
- t(s,T) : seed s의 execution trace와 UAF bug trace T간의 유사성 측정
- bug trace의 cover된 target 순서 고려 : P, 아니면 B
- 3가지 UAF event를 고려 : 3T
- target prefix $t_P(s,T)$ : s가 순차적으로 T에 있는 site cover
- UAF prefix $t_{3TP}(s,T)$ : s가 순차적으로 T의 UAF event cover
- target bag $t_B(s,T)$ : s가 T에 있는 site cover
- UAF bag $t_{3TB}(s,T)$ : s가 T의 UAF event cover
### 4.2.3. Combining Target Similarity Metrics
- 정밀한 metric P를 사용하는 것은 target으로의 direction을 잘 평가해줌
- P의 target bug trace와 일치하는 seed를 구별 할 수 있지만 다른 metricdms qnfrk
- 덜 정밀한 metric은 정밀한 metric이 제공하지 않는 정보를 제공한다.
- P는 figure 2의 UUU와 UFU를 측정하진 않지만 B는 측정한다.

이를 모두 적용하기 위하여 사전순으로 다음과 같이 결합함

$t_{P-3TP-B}(s,T) := <t_P(s,T),t_{3TP}(s,T),t_B(s,T)>$

우선순위

1. prefix에서 많은 위치를 cover하는 seed를 우선시 함
2. 순차적으로 많은 UAF event에 도달
3. target에 가장 많이 도달하는 seed

우리는 기본적으로 P-3TP-B를 사용
## 4.3. UAF-based Distance
- seed distance는 DGF에서 중요
- 기존의 seed distance는 단일 위치에 대해서 계산
- UAF는 순서를 고려해야함
- 우리는 순서를 고려하는 seed distace matric을 제안
### 4.3.1. Zoom: Background on Seed Distance
- AFLGo : seed distance d(s,T)는 execution trace의 각 BB에 대한 기본 블록 거리의 산술 평균으로 정의됨
- HawkEye : CG에서 function간 거리를 계산, caller,callee 관계의 가중치를 두어 edge를 결정
### 4.3.2. Our UAF-based Seed Distance
- target site의 순서를 고려함
- alloc,free,use 순서를 거치는 path를 선호
- light-weight static analysis를 사용하여 event 사이에 가능성있는 CG네의 edge에 가중치를 감소시킴

> 방법

1. $R_{alloc}, R_{free}, R_{use}$를 계산함 : $f_a, f_b$ 사이의 call edge의 방향을 나타냄
2. 각 방향에 대해 edge를 통해 거쳐갈 수 있는 이벤트의 수를 계산
3. double free를 감지하기 위하여 두 free event를 도달하는 call edge도 함께 계산
4. edge가 cover하는 event 수에 기반하여 call edge의 가중치를 감소시킴
5. *Hawkeye*와 유사하게 edge 가중치를 수정함
 
![expression1,2]()

![figure5]()


- *AFLGo*와 유사하게 target에 도달하는 가장 짧은 경로를 선호함
## 4.4. Power Schedule
- *AFLGo*는 simultated annealing을 사용하여 target에 가까운 seed에 더 많은 energy 할당
- *HawkEye*는 trace distance와 function-level similarity를 기반으로 할당

우리의 새로운 power schedule은 다음과 같은 직관을 사용함
- seed가 더 가까운 경우
- seed가 target과 더 유사한 경우
- target이 중요한 code junction에서 더 나은 결정을 내리는 경우
### 4.4.1. Cut-edge Coverage Metric
- 세밀한 접근 방식 : target bug trace와 seed execution trace를 BB-level에서 비교 : 높은 runtime overhead 발생
- 거친 방식 : function-level similarity 측정 > 하지만 target function에 도달한다 하더라도 해당 BB에 도달하진 않음
- cut-edge coverage metric : 두 방식의 중간 지점에서 critical decision node에서만 edge level 측정

> Definition 2 : cut edge

두 BB (source, sink) 사이의 cut edge는 decision node에서 나가는 edge이다.

![algorithm3]()

- binary program과 예상 UAF bug trace가 주어졌을때 UAFUZZ에서 cut/non-cut edge를 식별하는 방법
- Flatting된 bug trace의 모든  consencutive node 사이의 cut edge를 식별하고 축적
- 해당 함수 진입 시점부터 call event까지 cut edge를 계산함
- flatting 때문에 동일한 함수 내의 다른 점 사이의 cut edge를 계산해야함


![algorithm4]()

- 단일 함수 내에서 cut edge를 계산하는 방법
- source-sink BB 사이의 conditional jump인 critical node를 수집 (data-flow analysis 사용)
- critical node의 outgoing edge에 대해 sink BB에 도달할 수 있는지 확인
- intra-procedual 이기에 inter-procedual CFG 불필요

더 많은 cut edge를 실행하는 입력이 target의 더 많은 위치를 커버할 가능성이 높다는 것

$E_cut(T)$ : UAF bug trace T가 주어진 프로그램의 모든 cut edge set 

s의 cut edge score $e_s(s,T)$를 다음과 같이 정의

![expression 3]()

- hit(e) : edge e가 실행되는 횟수
- δ : non-cut edge를 cover하는 것에 대한 페널티 가중치
- loop에 의한 path explosion을 방지하기 위하여 buketized를 사용함

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