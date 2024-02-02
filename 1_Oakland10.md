[All You Ever Wanted to Know About Dynamic Taint Analysis and Forward Symbolic Execution](https://users.ece.cmu.edu/~aavgerin/papers/Oakland10.pdf)
# 0. Abstract
Dynamic taint analysis & forward Symbolic execution을 이용하여 다음과 같은 일을 많이 수행함
    
    1. Input Filter Generation
    2. Test case generation
    3. Vulnerabiliry discovery

이와 관련하여 이 논문은 두가지 파트로 되어있음

    1. dynamic taint analysis와 forward symbolic execution을 수행할 수 있는 general language 제시
    2. 이를 구현함에 있어서 important implementation choices, common pitfalls, considerations 
   
# 1. Introduction
## Dynamic analysis
- Dynamic analysis는 actual excution에 대한 추론을 가능하게함
- 한번에 하나의 실행에 대해서 고려함
  
다음 두가지 사항이 일반적으로 사용됨
    
    1. dynamic taint analysis
        - 미리 정의된 taint sources (ex user input)가 미치는 영향을 관찰함
    2. forward symbolic execution
        - logical formula를 자동으로 형성하여 논리 축약을 실행함
  
이 두가지 분석은 다음과 같이 함께 사용되기도 한다.

### Unkown Vulnerability Detection
알려지지 않은 취약점을 탐지한다, code injection을 막을 수 있다.

### Automatic Input Filter Generation
입력에서 exploite을 탐지하고 제거하는 Input Filter를 자동으로 생성하는데 사용된다.

### Malware Analysis
Malware Binary를 통하여 정보가 어떻게 이동하는지 확인할 수 있다.

### Test Case Generation
프로그램을 테스트 하기 위한 입력을 자동으로 생성하는데 사용된다.

이를 실제로 구현화는 과정속에서  여러 한계점, 구현 요령, 타협점을 찾게 된다. 따라서 이 논문은 다음과 같은 내용을 소개한다.

    1. Dynamic taint Analysis 와 forward symbolic exceution을 공식화한다.
    2. 공식화한 내용을 바탕으로 implementation details, caveats, and choices 를 설명한다. 
# 2. A General Language

## Overview
dynamic taint analysis과 forward symbolilc execution은 는 특정 언어에 대해서 정의되어왔다. 

이 논문에서 SimpIL (**Simp**le **I**ntermediate **L**anguage)을 제시하고 있다. 

![table1]

이 언어는 다양한 프로그래밍 언어에 대한 컴파일러에서 사용하는 내부 표현을 표현할 수 있다.

이 언어는 numbered statements로 이루어져 있고 assignments, assertions, jumps, conditional jump로 이루어져 있다. 또한 get_input을 통하여 사용자의 입력을 받을 수 있다. 또한 32-bit정수에 대해서만 교려한다. 이를 확장하는것은 간단하다.


## Operational Semantics
Operation semantics는 해당 언어로 작성된 프로그램을 어떻게 실행할지 엄밀하게 정의한다. 이를 통하여 실행을 기반으로 정의되는 dynamic program analyses를 정의하는 방법이 된다.


![figure1]
프로그램을 실행할때 패턴 매칭을 통하여 statement에 맞는 applicable rule을 찾는다.

## Language Discussion
이 언어는 Dynamic taint analysis와 forward symbolic execution을 보이기 위하여 디자인 되었기 때문에 function, scope에 대한 구현을 하지 않았다. 이러한 생략은 다음과 같은 두가지 접근방법으로 간단하게 추가할 수 있다.

### 1. high-level 언어를 SimpIL로 컴파일
함수, 버퍼에 대한 추상화는 BAP, BitBlaze와 같은 도구를 이용하여 가능하다. 이전 다른 연구를 통하여 이를 증명하였다.

### 2. higher-level constructs를 SimpIL에 추가
예를들어 Call, RET 명령을 추가하여 직접 function을 지원할 수 있다. 

# 3. Dynamic Taint Analysis
Dynamic taint analysis는 sources(input)와 sinks(중요한곳) 사이의 흐름을 추적하는 것이다. taint source에서 파상된 데이터에 의존하는 모든 값은 taint되었다고 간주한다.

taint policy는 taint된 값이 실행중에 어떻게 이동하는지, 어떤 종류의 작업이 taint를 도입하는지, taint된 값에 대한 어떤 검사를 수행하는지를 의미한다.

이는 taint analysis application에 따라 다르다. 하지만 원론적인 concepts는 같다.

이와 관련하여 두가지 오류가 존재한다.

    1. Oveertainted
     - 값이 taint source에서 파생되지 않음에도 오염된것으로 판단함
    2. Undertainted
     - taint된 값임에도 불구하고 이를 탐지하지 못함

이 두 오류를 줄여야 한다.

## Dynamic Taint Analysis Sematntics
각 프로그램의 taint된 상태를 확인하기 위하여 우리의 언어의 값들을 다음과 같이 튜플 <v,τ> 로 재 정의한다. v는 기존 언어에서 의미하는 값이고 τ는 taint되어 있는지를 의미한다.

![table2]


아래와 같이 taint analysis policy를 확인한다.


![figure5]

## Dynamic Taint Policies
taint policty는 3가지 성질을 지정한다.

    1. 새로운 taint의 도입
    2. taint의 전파
    3. taint의 확인

### Taint Introduction
일반적으로 모든 변수, 메모리의 값이 초기화될때 taint되지 않았다고 가정한다. 

SimpIL에서 get_input이라는 하나의 input source만 가지고 있다.

하지만 실제 구현에서 여러가지 input source를 가지고 있다. 일반적으로 input source들을 구분한다. 예를들어 인터넷을 통해 들어오는 네트워크 입력은 taint를 도입할 수 있지만 신뢰할수 있는 파일에서 읽는 정보는 그렇지 않다.

특정 taint source를 독립적으로 추적할 수 있다.

### Taint Propagation
taint policy는 기존 데이터에섯 파생된 데이터의 오염상태를 지정한다. 예를 들어 t_1 or t_2는 t_1 또는 t_2가 taint 되어있을때 결과가 taint 되어있음을 의미한다.
### Taint Checking
값이 taint되었는지 확인하기 위하여 taint checking을 수행한다.
예를들어 P_gotocheck(t)는 해당 주소가 taint된 값을 가질때 jump를 수행하는것이 안전하면 T를 반환한다. 만약 F가 반환되면 프로그램이 비정상적으로 종료된다.

## A Typical Taint Policy
아래와 같은 typical attack detection policy를 tainted jump policy라고 한다.

![table3]

tainted jump policy는 control flow hijacking 공격을 방어하기 위함이다. 

주요 아이디어는 입력에서 파생된 값이 return address나 function pointer을 덮어쓰지 않게 한다. 즉 jump에 대한 안정성을 보장하는 것이다. 

이 를 구현하기 위하여 get_input에 의해 반환된 모든 값에 taint를 도입한다. 이후 taint된 값이 이항 연산등을 통하여 프로그램으로 직접적으로 퍼지게 된다.

### Different Policies for Different Applications
응용프로그램에 맞게 taint policy를 정해야 한다.
특히 Table III에 설명된 typical taint policy는 메모리 주소 오염에 대한 고려를 하지 못한다.


## Dynamic Taint Analysis Challenges and Opportunities

dynamic taint analysis와 관련하여 고려할 사항들이 있다.

### Tainted Addresses
메모리 연산은 메모리 셀의 주소와 해당 셀에 저장된 값이 관여한다.

Table III의 tainted jump policy는 이를 독립적으로 추적한다.

예를들어 다음과 같은 프로그램을 생각해보자.
``` 
    x := get_input(·)
    y := load(z+x)
    goto y
```
이와같은 코드는 모든 메모리 주소에 접근할 수 있다.

기존 tainted jump policy는 이를 탑지하지 못할 수 있다. 이를 해결하기 위하여 tainted address policy를 사용할 수 있다.

![table5]

물론 이와같이 확장할경우 overtaint의 위혐이 있다. 즉 jump에 대한 검사의 undertaint와 address에 대한 검사의 overtaint를 잘 조절해야한다.

### Control-flow taint
control-flow를 통하여 data flow가 변할 수 있다. 이를 고려하지 않으면 taint를 결정할 수 업시게 전체 분석이 undertaint될 수 있다. 불행히 순수한 dynamical taint analysis를 통해서는 contro-flow taint를 계산할 수 없다. 왜냐하면 단일 execution에서는 하나의 path만 실행되기 때문이다.

이 문제를 해결하기 위해선 다음과 같은 해결방법이 있다.

    1. static analysis 사용
     - static analysis를 통하여 control-flow를 계산할 수 있다.
    2. heuristics 사용
     - heuristics을 사용하여 상황에따라 overtaitn, undertaint할지 결정한다.

### Sanitization
Dynamic taint analysis는 taint를 추가 한다. (제거하지 않는다) 이는 프로그램이 실행될때 계속 더 많은 값이 taint 되는 taint spread 문제를 발생시킨다.

taint analysis에서는 값에서 taint를 제거하는것 또한 중요하다.
예를 들어서 b = a ^ a 는 b를 0으로 초기화 되는 의미이지만 taint하다고 판단할 수 있다. 이를 방지하여야 한다.

### Time of Detection vs Time of Attack
taint가 전파되는 시간과 그 taint가 발생하는 시간과의 차이가 존재한다. 
# 4. Forward Symbolic Execution
Forward Symbolic Execution은 다양한 입력에 따라 프로그램의 동작을 한번에 추론할 수 있도록 한다.
## Applications and Advantages
> Multiple Input

한번에 여러 입력에 대해 분석할 수 있다. 

## Semantics of Forward Symbolic Execution
일반 실행과 Forward Symbolic Execution의 차이점은 get_input()의 결과가 구체적인 값 대신 기호를 반환한다. 기호를 포함하는 표현식은 구체적인 값으로 평가될 수 없다. SimpIL언어를 Table VI와 같이 확장된다.

![table6]

## Forward Symbolic Execution Challenges and Opportunities
### Symbolic Memory Addresses
메모리에서 값을 저장하거나 불러올때 참조되는 주소가 구체적인 값이 아닌 사용자 입력에 파생된 표현식(기호)일때 발생하는 문제이다.

실제 프로그램에서도 사용자 입력에 따라 달라지는 테이블 조회 등 여러 형태로 빈번하게 발생하는 문제이다.

Symbolic Memory Addresses는 Single path에 대한 execution에서도 aliasing issues를 발생할 수 있다. 두 값이 동일한 주소를 참조할때 발생한다.

이를 해결하기위하여 메모리를 참조할때 두가지 방법이 사용된다

    1. Symbolic Memory Address를 제거하기 위하여 건전하지 않는 가정을 한다. 이 논문에서는 코드를 변형하였다.
    2. SMT Solver에서 이를 수행하도록 한다.
    3. 두 값이 동일한 주소를 가르키고 있는지 추론하기위하여 alias analysis를 수행한다.
### Path Selection
경로가 분기될때 어떠한 분기를 먼저 탐색할지 결정해야 한다. Smmbolic Execution을 수행할때 이를 tree로 생각할 수 있다. 분석은 tree의 root node에서 시작하여 fork가 발생할때 child node를 생성한다. 이때 다음 탐색할 node를 결정하는 것은 중요하다 왜냐하면 무한 루프에 빠질 수 있기 때문이다.

이와 같은 "stuck"인 상황을 회피하기 위하여 다음과 같은 방법을 사용할 수 있다.
#### Depth-First-Search
탐색시 maximum depth를 설정하여 무한 루프를 방지한다.
#### Concolic Testing
Concolic Testing은 구체적인 실행을 사용하여 프로그램 실행을 추적하는 것이다. Forward Symbolic Execution과 동일한 경로를 따르지만 조건을 선택하고 해당 조건문을 부정하여 다른 경로로 강제할 구체적 입력을 생성할 수 있다.
#### Random Paths
random하게 탐색을 할 경우 더 낮은 상태에 대해 더 높은 가중치를 부여하여 stuck을 방지할 수 있다.
#### Heuristics
아직 다루지 않은 코드에 도달할 가능성이 높은 상태에 더 많은 가중치를 준다. 실행되지 않은 명령어까지의 거리 등을 변수로 한다.
### Symbolic Jumps
GOTO 규칙은 LOAD및 STORE와 비슷하다. 하지만 Forward Symbolic Execution에서는 Jump target이 구체적인 위치가 아닌 표현식 일 수 있다. 이와같은 문제를 Symblic jump problem 이라고 한다.

이 문제는 많은 연구에서 직접적으로 다루지 않고 있다. 다음은 Symblic jump을 다루는 3가지 방법이다.

#### Use Concolic analysis
Concolic analysis를 사용하여 간접적으로 jump targets을 확인한다. concrete execution 에서 점프 대상이 확인되면 이를 symbolic execution에 적용하는 것이다.

단점은 알려진 jump targets만 탐색하기 때문에 프로그램의 전체를 탐색하기는 더 어려워진다.
#### Use SMT solver
SMT solver를 이용하여 변수에 대한 값 할당(구체적인 점프 대상)을 얻고 이 값을 부정하는 합집합을 다시 SMT solver에 요청한다. 효육적이지 못하다.

#### Use Static analysis
정적분석을 사용하여 전체 프로그램에 대한 추론과 가능한 jump target을 찾을 수 있다. Binary-level jump
static analyses는 해당 식에서 참조될 수 있는 값들에 대해 추론할 수 있다.
### Handling System and Library Calls
C 언어의 read와 같은 함수로 interupt가 발생할 수 있다. 이때 read가 새로운 기호 입력을 반환하게 하게 한다.

concolic 기반으로 접근한다면 간단하다는 장점을 갖고 있다.

구현하기 쉽고 프로그램이 환경과 상호작용하는 문제를 우회한다. 반면 완벅한 분석을 제공하지 않는다. 왜냐하면 일부 호출은 동일한 입력을 받더라도 같은 결과를 반환하지 않기 때문이다.


### Performance 
이 방법의 효율성에 대해서 생각해보면 
    
    1. exponential number of program branches
    2. exponential number of formulas
    3. exponentially sized formula per branch

각 분기지점에서 fork가 발생하기 때문에 expontential 하게 늘어난다. 이를 해결하기 위하여 다음과 같은 방법이 사용된다

- 더 좋은 하드웨어 사용
- 치환으로 expotential하지 않게 변형
- formula의 중복을 제거
- subformula에 대한 정보를 cache에 저장하여 재사용
-  weakest precondition 사용

### Mixed Execution
실제로 사용되는 프로그램의 유형에 따라 symbolic input을 특정 형식의 입력으로 제한할 수 있다.

일부 입력은 구체적으로 허용하고 일부 입력을 기호적으로 허용한 것을 mixed execution이라고 한다.

이는 구체적인 값에 관한 계산을 프로세서에서 수행할 수 있도록 한다. 즉 사용자 입력에 의존하지 않는 부분은 concrete execution의 속도로 실행할 수 있다.
# 5. Related Work
## Formalization and Systematization
Dynamic Security mechanisms은 새로운것은 아니지만 이전 연구들은 dynamic taint analysis와 forward symbolic execution을 형식화 하지 않았다.

이 논문은 프로그래밍 언어를 정의하여 의미론적 모호성을 해소하였다.
## Applications
- Automatic Test-case Generation
- Automatic Filter Generation
- Automatic Network Protocol Understanding
- Malware Analysis
- Web Applications
- Taint Performance & Frameworks
- Extensions to Taint Analysis
