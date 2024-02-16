[Directed Greybox Fuzzing](https://acmccs.github.io/papers/p2329-bohmeAemb.pdf)

# 0. Abstract
기존의 Graybox Fuzzer(GF)는 문제가 있는 변경이나 패치, 중요한 system call, 위험한 위치, stack trace에 있는 함수등 특정 방향으로 효율적으로 fuzzing할 수 없다.

이 논문에서는 주어진 프로그램의 위치에 효율적으로 도달하는 입력을 생성하는 Directed Graybox Fuzzing(DGF)를 소개한다.

대상 위치에 더 가까운 seed에 많은 에너지를 할당하고 멀리 있는 seed에 에너지를 줄이는 annealing-based power schedule을 개발하고 평가한다. 우리의 구현체인 *AFLGo*로 수행한 실험은 DGF가 symbolic execution 기반 whitebox fuzzing 및 undirected graybox fuzzing보다 우수하다는것을 보여준다. DGF를 patch testing, crash reproduction에 적용한 사례를 보여주고 *AFLGo*를 Google의 fuzzing platform *OSS-Fuzz*와 통합하는 방법을 논의한다. directed로 인하여 *AFLGo*는 *LibXML2*와 같이 잘 fuzzing 된 프로젝트에서 39개의 bug를 찾았고 17개의 CVE가 할당되었다.
# 1. Introduction
GF는 가벼운 instrumentation을 사용하여 아주 작은 성능 overhead로 input에 의해 실행되는 path에 대한 unique identifier을 결정한다.

새로운 input은 제공된 seed input을 mutation하여 새로운 path를 실행하는 경우 fuzzer의 queue에 추가된다. *AFL*은 수백개의 취약점을 발견하는데 기여하였고 허공에서 유효한 이미지 파일을 생성할 수 있다는 것이 입증되었고 이를 확장하는데 관여한 큰 커뮤니티를 가지고 있다.

기존 GF는 directed될 수 없다. undirected fuzzer와 달리 directed fuzzer는 대부분의 시간을 특정 대상 위치에 도달하는데 사용된다. directed fuzzer의 적용 사례는 다음과 같다.

1. patch testing 
   
   변경된 statment를 대상으로 설정하여 path testing을 수행하면 중요한 구성 요소가 변경될때 이것이 취약점을 도입하였는지 확인할 수 있다. 
2. crash reproduction
   
   stack trace의 method call을 대상으로 설정하여 crash reproduction을 수행할 수 있다. 현장에서 crash가 보고될때 stack trace와 일부 환경 매개변수만 내부 개발팀에 전송된다. 사용자의 개인정보 보호를 위하여 구체적인 crash input은 종종 사용될 수 없다. directed fuzzer는 내부 팀이 crash를 reproduction 할 수 있게 해준다.
3. static analysis report verification
   
   static analysis tool이 잠재적으로 위험하다고 보고하는 statement를 대상으로 설정하여 static analysis report verification을 수행한다. 
4. information flow detection

    민감한 source와 sink를 대상으로 설정하여 information flow detection을 수행한다. 데이터 유출 취약점을 노출하고자 하는 연구자는 개인정보를 포함하는 민감한 소스를 실행하고 데이터가 외부 세계에 보이는 민감한 sink를 실행하는 실행을 생성하려고 한다. directed fuzzer는 이러한 실행을 효율적으로 생성하는데 사용될 수 있다.


## 1.1. symbolic execution
기존 대부분의 directed fuzzer는 SE에 기반하고 있다. SE는 program analysis와 constraint solving을 사용하여 다른 path르 실행하는 입력을 생성하는 WF 기술이다. directed fuzzer를 구현하기 위하여 SE는 체계적인 path serach 때문에 선택되었다.

control-flow graph에서 대상 위치로의 path를 π라 할때 SE는 π를 실행하는 모든 입력에 만족하는 논리공식 φ(π)를 구성할 수 있다. SMT solver는 path contraint φ(π)를 만족하는 경우 path contraint에 대한 solution인 input t를 생성한다. 즉 t는 π를 실행하는 input 이다.

DSE는 도달 가능성 문제를 constraint satisfaction problem으로 다룬다. 대부분의 path가 실제로는 실행 불간읗 하기 때문에 검색은 일반적으로 중간 대상에 도달하는 실행 가능한 path를 반복적으로 찾는다. 

![figure1]()

예를들어 patch testing tool (Katch)는 SE engine (Klee)를 사용하여 변경된 문장에 도달한다. 위 그림의 1480 line을 대상으로 설정하고 Katch가 1465 line 까지 도달하는 경로 π0를 찾았다고 가정하자. Katch는 φ(π0)&(hbtype == TLS1_HB_REQUEST)를 constraint solver에 전달하여 실제로 1480 line을 실행하는 input을 생성한다. GF와 달리 SEBWF는 directed fuzzing을 구현하는 간단한 방법을 제공한다.

DSE는 program analysis와 constraint solving에 상당한 시간을 소비한다. 따라서 DSE가 하나의 input을 생성할때 GF는 수십배의 input을 생성할 수 있다. 

이 논문에서는 DGF를 소개한다. 도달 가능성 문제를 최적화 문제로 보고 생성된 seed가 대상까지 거리를 최소화 하기 위해 특정 meta-heuristic 을 사용하여 seed 거리를 계산하기 위해 각 basic block에서 대상까지 거리를 계측한다. seed distance는 procdural 하지만 우리의 새로운 측정은 call graph에서 한번, 각 CFG에서 한번만 분석gksek. runtime에서  fuzzer는 실행된 basic block의 거리의 평균으로 seed distance를 계산한다.

DGF가 seed 거리를 최소하기 위하여 mata-heuristic은 *simulated annealing*으로 불리면 power schedule로 구현된다. power schedule은 모든 seed의 energy를 제어하고 seed energy는 fuzzing하는데 사용된 시간을 지정한다. 다른 GF와 마찬가지로 analysis를 compile time으로 이동함으로서 runtime의 overhead를 최소화 한다.


## 1.2. directed graybox fuzzing
우리의 실험은 DGF가 DSE보다 효과성, 효율성 측면에서 우수함을 보여준다. 하지만 두 기술의 통합은 각 기술을 개별적으로 사용하는것보다 더 우수하다. 우리는 AFL에 DGF를 구현한 *AFLGo*를 만들었다. patch testing을 위하여 *AFLGo* 를 *Katch*와 비교하여 더 좋은 성능을 보였다.

crash reproduction을 위하여 *BugRedux*와 비교하여 3배 더 좋은 성능을 보였다.

우리의 실험은 simulated annealing 기반 power scheduling이 효과적이며 *AFLGo*가 효과적으로 directed 할 수 있음을 보여준다. AFL에 비해 AFLGo가 더 좋은 결과를 얻었다.

또한 AFLGo와 OSS-Fuzz를 통합하였는데 더 좋은 성능을 얻었다.
## 1.3. main contribution
- GF와 simulated annealing의 통합
- inter-rocedural distance의 측정
- *AFLGo*로 구현된 DGF
- *OSS-Fuzz*와 *AFLGo*의 통합
- DGF의 patch testing, crash reproduction 에서의 효과성에 대한 평가

# 2. Motivating Example
우리는 Heartbleed 취약점을 case study와 motivating sexample로 사용하여 DF에 대한 두가지 접근 방법을 논의한다. 전통적으로 DF는 SE에 기반한다. 우리는 SE engine *Klee*에 기반한 patch testing tool *Katch*와 이 논문에서 제시된 DGF 구현체 *AFLGo*를 비교한다.

## 2.1. Heartbleed and Patch Testing
Heartbleed (CVE-2014-0160)은 명빅히 안전한 protcol (SSL/TLS)을 통해 전송된 데이터의 개인정보 및 무결성을 위협하는 취약점이다. 이것은 man in the middle (MITM) 없이 공격 가능하다. 이 버그는 OpenSSL library에 도입된 2년이 지난 후에 탐지되었으며 광범위하게 분포가 발생하였다.

2011 새해 첫날 새로운 기능을 구현한 commit 4817504d에 의해 도입되엇다. 변경된 문장들을 대상 위치로 삼는 DF가 이를 발견하였다면 광범위한 확산을 막을 수 있었을 것이다. 현재 *OpenSS*L은 50만줄의 코드로 이루어져 있다. 해당 commit은 500줄 이상의 코드를 추가하였다. 실제로 최근 변경사항만이 오류를 발생하기 쉬운 것으로 간주되는 상황에서 OpenSSL 전체를 undirected fuzzing을 수행하는것은 어려울 것이다. DF는 이러한 변경사항을 훨씬 더 효율적으로 실행할 것이다. 대부분의 patch testing tool (*Katch*, *PRV*, *MATRIX*, *CIE*, *DiSE*, *Qietal.`s*)등은 SE를 기반으로 한다. 이중 *Katch*를 대표로 비교하였다.

## 2.2.  Fuzzing the Heartbleed-Introducing Source Code Commit
### 2.2.1. Katch
*Katch*는 *Klee* 위에서 구현된 patch testing tool이자 DSE engine 이다. 먼저 *OpenSSL*은 *LLVM 2.9*로 컴파일 되어야 한다. *Katch*는 한번에 변경된 basic block 하나를 처리한다. 우리의 regression test suite에 의해 cover되지 않은 도달 가능한 대상위치로 11개의 변경된 basic block을 식별한다. 각 대상 `t`에 대해 *Katch*는 다음과 같은 *greedy search*를 수행한다. 
1. *Katch*는 regression test suite에서 `t`에 가장 가까운 seed `s`를 식별한다.
2. program analysis를 사용하여 `t`에서 가장 가까운 실행 분기 `b_i` 를 식별하고 `s`가 실행하는 모든 분기 조건의 연속인 path constraint `Π(s) = φ(b0)∧φ(b1)∧..∧φ(bi)∧...` 를 구성한다.
3. `b_i`를 부정하기 위해 수정해야 하는 `s`의 구체적인 input byte를 식별한다.
4. `b_i`의 조건을 부정하는 `Π'(s) = φ(b0)∧φ(b1)∧..∧¬φ(bi)`을 도출하고 이를 *Z3*(SMT solver)를 이용하여 concrete 한 input byte를 계산한다.


### 2.2.2. Challenge

DSE 기반 WF는 효과적이지만 비용이 매우 많이 든다. 우리의 실험해서 *Katch*는 24시간 동안 heartbleed를 탐지하지 못하였다. 탐색의 모든 새로운 path에 대해 distance가 runtime에 다시 계산된다.interpreter가 모든 byte code를 지원하지 않고 path constraint가 floating point arithmetic과 같은 모든 언어 기능을 지원하지 않을 수 있기에 검색이 불완전할 수 있다. 왜냐하면 sequential search로 인하여 매 대상마다 검색이 새로 시작된다.

### 2.2.3. Opportunities

# 3. Technique

## 3.1. Greybox Fuzzing

## 3.2. A Measure of Distance between a Seed Input and Multiple Target Locations

## 3.3. Annealing-based Power Schedules

## 3.4. Scalability of Directed Greybox Fuzzing

# 4. Evaluation Setup

## 4.1. Implementation

## 4.2. Infrastructure

# 5. Application 1 : Path Testing

## 5.1. Patch Coverage

## 5.2. Vulnerability Detection

# 6. Application 2 : Continuous Fuzzing

## 6.1. LibXML2

## 6.2. LibMing

# 7. Application 3 : Crash Reproduction

## 7.1. Is DGF Really Directed?

## 7.2. Does DGF Outperform the Symbolic Execution-based State of the Art?

# 8. Threats to Validity

# 9. Related Work

# 10. Conclusion
*AFL*과 같은 Coverage-based GF는 프로그램 analysis의 부담 없이 많은 path를 cover하려고 한다. *Klee*, *Katch*와 같은 SE 기반 WF는 필요할때 symblic program analysis와 constraint solving을 사용하여 검색을 정확하게 지시한다.

SE는 directed fuzzer를 구현하기 위한 기술로 선택되어 왔다. 주어진 대상에 도달하는 것은 path constraint를 해결하는 문제이다. SE는 특정 path를 탐색하기 위하여 수학적으로 엄밀한 framework를 제공한다. 반면 GF는 본질적으로 무작위적이며 box 밖에서 direction을 지원하지 않는다. 근본적으로 GF도 단지 random seed file 에서 random mutation을 적용할 뿐이다.

이 논문에서 이러한 direction을 GF에 도입하려고 시도하였다. runtime에서 GF의 효율성을 유지하기 위해 대부분의 프로그램 분석을 계측시간으로 옮기고 test generation중에 simulated annealing으로 global meta-heuristic을 사용한다. DGF는 DSE와 달리 무거운 프로그램을 분석할때 runtime overhead나 instruction을 path constraint로 인코딩 하지 않는다.

우리의 GF인 *AFLGo*는 단 몇천줄의 코드로 구현되었고 설정이 쉽다. SE와 달리 DGF는 무작위적이며 주어진 목표로 체계적으로 유도될 수 없다. 따라서 *AFLGo*가 기존의 DSE 기반 WF보다 성능이 좋은것은 놀라운 일이다. 효과성(더 많은 취약점), 효율성(같은 시간 더 많은 목표) 측면에서 모두 우수하다.

DGF는 다양한 방법으로 사용될 수 있다.
- 문제가 되는 변경이나 패치
- 중요한 system call
- 보고된 취약성의 stack trace
- crash 재현
  
미래의 작업으로 SEBBWF과 DGF를 통합할 것이다. SE는 최적화 문제로 serach를 모두 활용하는 통합된 DF은 그들이 결합된 강점을 활용하여 개별적인 약점을 완화할 수 있을 것이다. 실험에서 입증된것과 같이 잠재적으로 취약한 프로그램 변경을 밝혀내기 위한 더 효과적인 patch testing으로 이어진다.

우리는 위험한 위치나 보안에 중요한 구성 요소를 지적하는 static analysis tool과 통합될때 *AFLGo*의 효과성을 평가할 것이다.



# 11. 의문점
만약 commit된 부분으로 인하여 기존 코드에서 발생하는 취약점을 탐지하는 방법?