[Evaluating Fuzz Testing](https://cseweb.ucsd.edu/~dstefan/cse227-spring20/papers/klees:evaluating.pdf)

# 0. Abstract
Fuzzing은 소프트웨어에서 중요한 버그를 발견하는 좋은 전략이다. 최근 연구자들은 새로운 fuzzing 기술, 전략, 알고리즘을 고안하여 실험적으로 평가하였다. 이 논문에서는 실험적으로 평가하는 과정에서 어떤 experimental setup을 통하여 신뢰할 수 있는 결과를 얻을 수 있는지에 대해 다룬다.

최근 32의 fuzzing paper이 수행한 실험적 평가(experimental evaluation)를 통해 모든 evaluation에서 문제를 발견하였다. 이후 기존의 fuzzer를 사용하여 자체적인 광범위한 experimental evaluation을 수행하였다.

우리의 결과는 기존 experimental evaluation에서 발견한 general problem이 실제로 잘못되거나 오해의 소지가 있는 evaluation으로 이어질 수 있음을 보여주었다.

우리는 보고된 결과를 더 견고하게 만들기 위하여 fuzz testing alogrithm의 experimental evaluation을 개선하는데 도움이 될 몇가지 지침을 제시하였다.

# 1. Introduction
## 1.1. fuzz 소개
fuzzer는 반복적으로 random input을 생성하여 대상 프로그램을 테스트하는 도구이다. SMT(Satisfiability Modulo Theories) solver, symbolic execution, static analysis를 포함하는 더 정교한 도구보다 단순하지만 놀랍도록 효과적이다.

예를들어 인기있는 fuzzer인 *AFL*은 수백개의 버그를 찾는데 사용되었다. symbolic executor *angr*와 비교하였을대 AFL은 같은 대상에서 76& 더 많은 버그를 발견하였다.

fuzzer가 외 효과적인지에 대한 평가는 주로 실험적 평가로 이루어진다. 새로운 아이디어에 대한 영감이 수학적인 분석에서 비롯될 수 있지만 평가는 실험적으로 진행된다. 연구자가 새로운 fuzz algorithm(A)을 개발할때 그것들이 현재 다른 fuzzer에 비해 어떤 이점을 제공하는지 실증적으로 보여주어야 하기때문에 다음을 시용한다.

## 1.2. 평가 기준
- baseline fuzzer B 와 비교
- *banchmark suite*라는 샘플 프로그램
- *banchmark suite*에서 A와 B를 실행할때 측정하는 성능 지표 (충돌 입력에 의해 식별된 버그의 수)
- meaningful set of *configuration parameters* (seed file, timeout 등)

평가는 fuzzing의 근본적인 성격인 무작위성을 고려해야한다. 평가 대상 프로그램에 대한 각 fuzz execution은 무작위성으로 인하여 다른 결과를 낼 수 있다. 따라서 fuzzer의 성능을 평가할때 전체 분포를 샘플링 하기 위해 충분히 많은 시도를 측정해야 하므로 검정을 사용하여 A가 B에 비에 개선된 측면이 실제 성능인지, 우연으로 인한것인지 결졍해야 한다.

이러한 단계중 하나를 수행하지 않거나 권장사항을 따르지 않으면 오해의 소지가 있거나 잘못된 결론을 낼 수 있다. 

## 1.3. 실험
우리는 최근에 발표된 fuzz testing에 대한 32편의 논문을 검토하여 그것의 "실험적 평가"를 연구했다.

위에서 언급한 모든 단계를 제대로 수행하는 fuzz testing evaluation이 없다는 것을 발견하였다. 우리는 AFLFast (A) AFL (B)를 이용하였다. 왜냐하면 32편의 논문중 14편에서 AFL을 기준으로 평가하였기 때문이다.

또한 3개의 binutils program(nm, objdump. cxxfit), 2개의 이미지 처리 프로그램 (gif2png, FFmpeg)를 대상으로 평가하여 위의 레시피에서 벗어난 실험은 쉽게 잘못된 결론을 도출할 수 있음을 발견하였다.

## 1.4. 이유
### 1.4.1. Fuzzing performance under the same configuration can vary substantially from run to run
검토한 논문의 2/3가 단일 실행을 비교한다. 이는 full picture를 제공하지 않는다. 예를 들어 *nm*에서 한번의 *AFL* 실행은 1200개의 crash input을 발견하였지만 *AFLFast*의 경우 800개를 발견하였다. 하지만 30회 실행을 통하여 중앙 값을 비교하면 *AFL
* : 400, *AFLFast* : 1250회 의 crash를 발견하였다.

평균을 비교하는것도 충분하지 않다. 우리는 통계적 검정을 통하여 어떤 경우에서는 성능에서 명백한 차이가 통계적으로 유의미하지 않다는 것을 발견하였다.

### 1.4.2. Fuzzing performance can vary over the course of a run
짧은 timeout (16시간 미만)의 실험은 11개의 논문에서 발견되었고 오해의 소지가 있는 picture가 나올 수 있다.

empty seed in *gif2png*

||*AFL*|*AFLFast*|
|:---:|:---:|:---:|
|13hr|0|40|
|24hr|39|52|

non-empty seed in *nm*에서도 6시간 까지는 *AFL*이 통계적으로 유의미하게 우수한 성능을 보였지만 24시간 이후에는 추세가 반전되었다.

### 1.4.3. substantial performance variations based on the seeds used
empty seed를 사용한 *AFLFast*는 *nm*에서 1000개 이상의 충돌을 발견하였지만 작은 non-empty seed를 사용할때는 24개를 발견하였고 *AFL*이 발견한 23개와 크게 다르지 않다.

그럼에도 불구하고 대부분의 논문들에선 seed selection을 가볍게 처리하였으며 어떤 시드든 동일하게 잘 작동할것이라고 가정하는 것처럼 보였지만 구체적인 내용을 제공하지 않는다.

## 1.5. 성능 측정
14개의 논문은 code coeverage를 사용하여 fuzzing 효과를 평가하였다. 더 많은 코드를 커버하는 것은 직관적으로 더 많은 버그를 찾는 것과 관련이 있다고 여겨진다. 그래서 가치가 있어 보이지만 이러한 correlation은 약할 수 있으며 찾아낸 버그의 수를 직접 측정하는것이 더 선호된다.

그럼에도 직접 측정을 사용한 논문은 1/4에 불과하다. 대부분의 논문은 찾아낸 crash의 수를 세고 같은 버그를 트리거하는 input이 중복을 제거하기 위한 hueristic procedure를 시도하였다.

가장 인기있는 두 추론 방법은 *AFL*의 커버리지 프로필, (fuzzy) stack hash를 사용되었다. 하지만 이러한 중복 제거 hueristic은 비효율적일 수 있다.

## 1.6. 추가 실험
추가 실험에서 ground truth를 계산하였다. *cxxfilt*에 모든 패치를 적용하였다. 우리는 특정 패치로 인해 우하하게 종료되도록 만든 모든 input을 그룹화 하였다.

이는 패치가 단일 개념적 버그 수정을 대표한다는 것을 확인하였다.

coverage profile에 의해 "unique"하다고 간주되는 57142개의 충돌 입력이 9개의 구별된 패치에 의해 해결되었다는것을 발견하였다. 이는 버그 수의 극적인 과대계산을 나타낸다.

*AFLFast*는 *AFL*보다 훨씬 더 많은 unique crash input을 찾았지만 주어진 실행에서 더 많은 unique한 bug를 찾을 가능성은 약간 높을 뿐이다.

*stack hash*는 더 나은 성능을 보였지만 여전히 버그를 과대 계산하였다. 주어진 시험에서 버그가 대략 500개의 *AFL coverage-unique crash*에 mapping되는 대신 46개의 *stack hash*에 mapping되었다. 또한 false negative의 영향을 받아서 한 버그로부터 crash에 대한 해시의 대략 16%가 다른 버그로부터의 crash에 의해 공유되었다. 5가지 경우에서 unique bug는 오직 하나의 crash에 의해 발견되었고 그 충돌은 unique하지 않은 hash를 가지고 있다. 이런 독특한 버그가 "de-duplication"에 의해 삭제되었음을 의미한다.

이 실험은 이 분야에서 가장 실질적인 실험중 하나로 성능을 평가하기 위하여 heuristic에 의존하는것이 현명하지 않음을 시사한다. 더 나은 접근방식은 우리가 위에서 한것처럼 bug에 대한 fuzzer를 평가하거나 우리가 검통한 6개의 논문에서 수행된것처럼 CGC나 LAVA와 같은 synthetic suite를 사용하여 ground truth에 직접 대응하여 측정하는 것이다.

## 1.7. banchmark suite
전반적으로 fuzzing 성능은 대상 프로그램에 따라 달라질 수 있으므로 다양하고 대표적인 *banchmark suite*에서 평가하는것이 중요하다. 우리의 실험에서 *AFLFast*가 binutils 프로그램에서 *AFL*보다 더 나은 성능을 보인다는것을 발견하였다. 하지만 이미지 처리 프로그램에서 통계적으로 유의미한 이점을 제공하지 않았다. 이 프로그램들이 그 평가에 포함되었다면 논문의 득자들은 그 이점에 대해 더 미묘한 결론을 내렸을 것이다.

일반적으로 소수의 논문만 공통적이고 다양한 *banchmark suite*를 사용하였다. 6개가 *CGC*나 *LAVA-M*을 사용하였고 2개는 real-world 프로그램에서 논의하였고 나머지는 소수의 선별된 프로그램을 사용하였다. 이러한 선택은 논문간에 겹치는 부분이 거의 없었다.

평가에서 사용된 real-world 프로그램의 중앙값은 7개였고 가장 흔한 프로그램 (binutils)은 4편의 논문에서만 공유되었다. 결과적으로 개별 평가는 내부적으로 오해의 소지가 있는 결론을 제시할 수 있으며 논문간에 결과를 비교하기 어렵습니다.

우리의 연구에서 fuzzing에 대한 의미 있는 과학적 진보를 위해서는 알고리즘 개선에 대한 주장이 더 확실한 증거에 의해 지지되어야 한다는 것을 제안한다. 32편의 논문에서 모든 평가는 이와 관련하여 부족하다.

이 논문에서는 미래의 논문의 평가가 따라야할 몇가지 명확한 지침을 제안한다. 연구자들은 [여러번의 실험을 수행하고 통계적 검정을 사용해야 한다](#4-statistically-sound-comparisons). 또한 [다양한 시드를 평가해야하고,](#5-seed-selection) [더 긴 timeout을 고려해야 한다.](#6-timeouts) [unique crash와 같은 heuristic이 아닌 ground truth에 기반하여 평가해야 한다.](#7-performance-meaasures) 마지막으로 [좋은 fuzzing benchmark 설정과 채택을 주장해야 한다.](#8-target-programs) 많은 노문에서 특정 목표를 선택하고 논문마다 그것을 바꾸는 관행이 있다. 잘 설계되고 합의된 benchmark는 이 문제를 해결할 것이다.

또한 de-duplication hueristic의 설정과 SAT solving과 같ㅌ은 영역에서 아이디어를 포함하고 우리의 결과가 연구 가치가 있다고 제안하는 다른 문제를 식별한다.


# 2. Background
fuzzing을 묘사될 수 있는 다양한 dynamic analysis가 있다. fuzzer의 통합적 특징은 concrete input에 의해 동작하고 그러한 input을 생성한다는 것이다. 그렇지 않으면 fuzzer는 다른 많은 design setting과 많은 argument setting으로 구현될 수 있다.

이 섹션에서 fuzzer가 어떻게 작동하는지 기본 사항을 설명하고 fuzzing evaluation에 대한 우리의 연구의 핵심을 형성하는 32편의 논문의 진보에 대해 언급한다.


## 2.1. Fuzzing Procedure
대부분의 fuzzer는 다음과 같은 절차를 따른다.

``` C
corpus ← initSeedCorpus()
queue ← ∅
observations ← ∅
while ¬isDone(observations,queue) do
    candidate ← choose(queue, observations)
    mutated ← mutate(candidate,observations)
    observation ← eval(mutated)
    if isInteresting(observation,observations) then
        queue ← queue ∪ mutated
        observations ← observations ∪ observation
    end if
end while
```

각 변수에 대한 설명은 다음과 같다.

    - initSeedCorpus: seed corpus 초기화
    - isDone: timeout 또는 goal을 달성하여 fuzzing을 정지해야 하는지 판단한다.
    - choose: seed로부터 mutation할 대상을 하나 뽑는다.
    - mutate: seed와 관찰한 것으로부터 새로운 seed를 생성한다.
    - eval: 관찰을 통하여 seed에 대해 평가한다.
    - isInteresting: seed에 대한 평가에서 input을 보존해야 하는지 여부를 결정한다.

seed corpus를 선택함으로 시작된다. 이 input을 반복적으로 mutation 하며 테스트중인 프로그램을 평가한다. matate input을 미래에 사용하기위하여 보관하고 관찰된 내용을 기록한다. fuzzer는 특정 목표(버그 찾기)에 도달하거나 timeout에 이르러 멈춘다.

다양한 fuzzer는 프로그램을 실행할때 관찰을 기록한다. *blackbox fuzzer*는 프로그램이 충돌했는지 여부를 관찰한다. *graybox fuzzer*는 실행중에 발생하는 중간 정보 (basic block, branch와 같은) 또한 관찰에 포함된다. *whitebox fuzzer*는 프로그램 source(or binary) code의 의미를 이용하여 관찰과 수정을 할 수 있으며 이는 복잡한 heuristic을 포함할 수 있다. 다양한 fuzzer는 더 나은 효율을 위하여 높은 overhead를 희생하며 다른 선택을 한다.

fuzzer의 궁국적인 목표는 crash를 발생시키는 input을 생성하는 것이다. 일부 fuzzer 구성에서 *isDone*은 queue를 확인하여 crash가 발생하는지 확인하고 crash가 발생한다면 loop를 중단한다. 다른 구성에서는 가능한 많은 crash를 수집하려고 시도하므로 crash 이후에 멈추지 않는다. 예를들어 *libfuzzer*는 crash을 발견하면 멈추지만 *AFL*은 계속하여 새로운 crash를 발견하려고 시도한다. 더 긴 실행 시간과 같은 다른 유형의 관찰도 바람직하며 이는 알고리즘 복잡성 취약점(algorithm complexity vulunerabilitie)를 나타낼 수 있다. 이러한 경우에 fuzzer의 출력은 개발자가 관찰을 확인, 재현, 디버그 할 수 있도록 퍼저 외부에서 사용할 수 있는 concrete input과 configuration 이다.

## 2.2. Recent Advances in Fuzzing
fuzz testing의 효과성에 대한 많은 연구가 진행되었다. 우리는 2012~18년 사이 fuzzing algorithm을 개선한 논문 32편을 찾아 분석하였다.

![table1]()

위 표는 논문을 연대순으로 나열하여 개선하려는 목표에 따라 요약하였다. 우리의 관심사는 이 논문들이 주장하는 개선을 어떻게 평가하는지에 있다.

### 2.2.1. initSeedCorpus
*Skyfire*, *Orthrus*는 프로그램에 대한 사전 분석을 실행하여corpus를 생성하고 mutator를 지원하기 위한 정보를 스스로 지원하는 방식으로 input seed selection을 개선하고자 제안한다. *QuickFuzz*는 유효하거나 흥미로운 입력의 구조를 지정하는 문법을 사용하여 seed selection을 지원한다. *DIFUZE*는 device driver로부터 input의 structure를 식별하기 위하여 up-front static analysis를 수행한다.
### 2.2.2. mutate
*SYMFUZZ*는 symbolic executor를 사용하여 seed를 mutate할 bit 수를 결정한다. 여러 다른 연구들은 프로그램에 대한 taint level observation을 인식하도록 mutate하며 특히 프로그램이 사용하는 input을 변형한다. 여러 다른 fuzzer들이 bit flipping, rand replacemnet와 같은 pre-defined data mutation 전략을 사용하는 곳에서 *MutaGen*은 dynamical slicing을 통하여 input을 parsing하는 mutator로 이용한다.

*SDF*는 seed 자체 속성을 사용하여 muation을 진행한다 대때로 grammar를 사용하기도 한다. *Chizpurfle*의 muatator는 안드로이드 시스템 서비스의 in-process fuzzing을 지원하기 위하여 java-level language construct에 대한 지식을 활용한다.

### 2.2.3. eval
*Driller*와 *MAYHEM*은 프로그램의 conditional guards가 bruteforce guessing을 통하여 만족시키기 힘들다는 점을 확인하여 때때로 eval 단계에서 symbolic executor를 호출하여 이를 해결한다. *S2F*도 동일하다.

다른 연구는 OS에 변경을 가하거나 다른 low level primitive를 사용하여 eval의 속도를 증가시킨다. *T-Fuzz*는 새로운 코드에 도달하는것을 방지하는 입력에 대한 검사를 제거하기 위하여 프로그램을 변환한다. *MEDS*는 퍼징 동안 오류를 탐지하기 위해 더 세분화된 run time analysis를 수행한다.
### 2.2.4. inIntersting
대부분의 논문이 crash에 초점을 맞추지만 일부 연구는 더 긴 실행시간이나 특별한 행동을 관찰한다. 

*Steelix*와 *Angora*는 조건을 만족시키는 방향으로 진행에 대한 더 셈리한 정보가 observation을 통해 나타나도록 계측한다.

*Dowser*와 *VUzzer*는 static analysis를 사용하여 프로그램 포인트에 서로 다른 보상을 할당한다. 이는 그 포인트를 통과하는 것이 취약점으로 이어질 가능성에 대한 추정에 기반하거나 CFG에서 더 깊은 포인트에 도달하기 위함이다.
### 2.2.5. choose
여러 연구는 특정 프로그램 영역에 도달하는지 여부를 기반하여 다음 input 후보를 선택한다. 후보 seed를 선택하기 위한 다양한 알고리즘을 탐구한다. 


# 3. Overview and Experimental setup
이 논문은 fuzz testing algorithm을 평가하는 기존의 연구 관행을 평가한다. fuzz testing algorithm A를 평가하는 것은 여러 단계를 필요로 한다.

- 비교 대상이 될 baseline algorithm B 선택
- 테스트할 대표적인 프로그램 집합 선택
- A,B의 성능을 측정하는 방법 선택
- 시드 파일을 선택, timeout 등 algorithm paramater
- A.B 모두 여러번 실행을 하여 그들의 성능을 통계적으로 비교

fuzz testing에 대한 논문들은 이러한 단계를 수행하는 과정에 상당한 차이가 존재한다. Table1은 32개 논문에 대해서 평가한다. 각 항목은 다음과 같다.

- benchmark : 테스트를 진행할 프로그램
- baseline : 기준 fuzzer
- trials : 시험 횟수
- variance : 성능의 변동성 고려 하였는지
- crash : crash가 bug로 어떻게 mapping 되어있는지
- coverage : coverage가 측정 되었는지
- seed : seed 파일을 어떻게 선택하였는지
- timeout 각 시험의 timeout이 얼마인지

이러한 평가들중 어떤것이 기술적 발전을 뒷받침하는 증거의 의미로 "좋은"것일까? 다음 절에서는 평가를 이론적, 경헙적으로 평가하여 부적절한 선택이 algorithm의 적합성에 대해 오해를 불러일으키거나 잘못된 결론을 내리게 할 수 있는 방법을 보여주는 실험을 수행한다. 어떤 경우에는 평가에 대한 "최선"의 선택이 열린 질문이라고 생각하지만, 특정 접근 방식이 취해져야 한다고 생각한다.(적어도 특정 방법은 취하지 않아야 한다.) 이 절은 우리 자신의 실험을 위한 설정을 설명한다.

## 3.1. Fuzzer
우리는 baseline fuzzer B 롤 AFL 2.43b를 사용하고 advanced algorithm fuzzer로 A를 선택하였다. AFLFast의 성능을 테스트 결과를 재현하려는 것이 아닌 실증적으로 fuzzer를 평가하는 접근 방식의 (무)타당성을 고려하기위하여 선택하였다. 또한 AFL은 오픈 소스이며 구축하기 쉽고 쉽게 비교할 수 있다. 또한 AFLNaive 설정을 사용하여 coverage tracking을 하지 않고 blackbox fuzzer로 사용할 수 있다.

### 3.2. Benchmark programs
우리는 다음 프로그램을 benchmark에 사용하였다.

*nm*, *objdump*, *cxxfilt*, *gif2png*, *FFmpeg* 

이것이 완벽한 benchmark suite는 아니다. (실제로 좋은 benchmark suite를 얻는 것은 미해결 문제라고 생각한다) 단지 이 프로그램들을 사용하여 다른 대상들에 대한 테스팅이 어떻게 다른 결론으로 이끌 수 있는지 보여주기 위하여 사용한다.

### 3.3. Performance measure
우리는 실험을 위해 특정 기간동안 fuzzer가 찾을 수 있는 "unique crash" 의 수를 측정하였다. 여기서 고유성은 AFL coverage 개념에 의해 결정된다. 특히 두 crash input은 같은 coverage profile을 가질 경우 같은 것으로 간주한다. 이 측정 방법은 흔하지 않지만 문제가 있다. [#7](#7-performance-meaasures)에서 다룬다.


# 4. Statistically Sound Comparisons
현대의 fuzzing algorithm을 testing 할때 근본적으로 mutation을 주목할 수 있지만 때로는 다른 방식으로 무작위성을 활용한다. 따라서 단순휘 fuzzer A와 baseline B를 각각 한번 실행하고 그것들의 성능을 비교하는것은 충분하지 않다. A,B 모두 많은 실험을 거쳐야 하며 그것들의 성능 차이를 판단해야 한다. 

하지만 대부분의 논문 (18/32)에서는 수행된 시험의 수에 대해 언급하지 않는다. 문맥에서 단서를 바탕으로 그들이 각각 한번의 시험을 수행했다는것을 알 수 있었다. 한가지 가능한 정당화는 무작위성의 "균등하게" 분포한다는 것이다. 즉 오래 실행하면 무작위 선택이 수렴하여 fuzzer가 동일한 수의 input을 찾게 될 것이다. 하지만 우리는 실험을 통하여 이것이 사실이 아님을 알아내었다. 즉 fuzzing 성능은 실행마다 극적으로 변할 수있다.

![figure2]()

figure 2를 보면 empty seed로 시작하는 AFL과 AFLFast의 시간에 따른 crash의 수를 그래프로 나타내었다. 각 plot에서 실선은 30번 실행에서의 중앙값을 나타내며 점선은 95% 신뢰구간과 최대,최소 결과를 나타낸다. 이를 분석해 본다면 단 하나의 실행을 고려하는것이 잘못된 결론으로 이끌 수 있음이 분명해진다. 

여러 실험을 수행하고 평균을 보고하는것이 더 나은방법이겠지만 변동성을 고려하지 않는것도 문제가 된다. 여러번 시험을 고려한 14개 논문중 11개는 성능의 변동성을 특정짓지 않았다. 대신 각각의 논문은 결론을 도출할때 A,B의 평균 성능을 비교하였다.

문제는 변동성이 충분히 높다면 평균의 차이가 통계적으로 유의미하지 않을 수 있다. 이러한 문제의 해결책으로 통계적 검정을 사용하는 것이다. 즉 성능의 차이가 우연이 아니라 실제로 존재할 가능성을 나타낸다. *Arcuri*와 *Braind*는 fuzzer와 같은 randome testing algorithm에 대하여 *Mann Whitney U-test*를 사용하여 A와 B의 확률적 순위를 결졍해야한다고 제안한다. 이 방법은 non parametric 이다.

다시 실험으로 돌아가서 단순히 평균을 비교하는것이 잘못된 결론을 낼 수 있음을 확인할 수 있다. *gif2png*의 경우 24시간 후 *AFLFast*는 51, *AFL*은 39건의 중앙값을 갖는다. 하지만 *Mann Whitney* 검정을 사용하면 통계적으로 유의미한 차이가 없음을 확인할 수 있다. 

table1의 variance 열에 C가 있는 3편의 논문은 평균과 함꼐 신뢰 구간을 제시하였다. 하지만 여기에서도 접근 방식의 성능을 기준과 통계적으로 비교하는 것이 아닌 독자가 시각적으로 판단하도록 한다.

> Discussion

통계적 검정을 사용하는 것은 논란의 여지가 없지만 최선의 검정 방법에 대해선 추가적 논의가 있을 수 있다. 특히 두가지 실행 가능한 대안은 *premutation test*과 *bootstrap-based test*이다. 이러한 방법들이 *Mann Whitney* 보다 적절한지 여부는 *Arcuri*와 *Briand*를 따른다.

Fuzzer A의 중앙값이 Fuzzer B보다 놓다는 것을 경정하는것도 중요하지만 effect size도 중요하다. A가 B보다 더 낫다고 해도 얼마나 더 좋은지는 알려주지 않는다. 우리는 측정된 중앙값의 차이를 살펴본다. 통계적 방법은 이 차이가 진짜 차이를 나타낼 가능성을 결정할 수 있다. *Arcuri*와 *Briand*는 *Vargha*와 *Delaney*의 A_12 통계를 제안한다.


# 5. Seed Selection

# 6. Timeouts

# 7. Performance Meaasures

## 7.1. Ground Truth: Bug founds

## 7.2. AFL Coverage Profile

## 7.3. Stack Hashes

## 7.4. Code Coverage

# 8. Target Programs

## 8.1. Real programs

## 8.2. Suites of artificial programs (or bugs)

## 8.3. Toward a Fuzzing Benchmark Suite

# 9. Conclusions and Future Work
