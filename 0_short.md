# Abstract

## Penetrating conditionals
### REDQUEEN: Fuzzing with Input-to-State Correspondence
[Link](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04A-2_Aschermann_paper.pdf)

최근 몇년간 fuzz 기반 automated software testing이 대두되고 있다. 특히 feedback-driven fuzz testing은 magic byte와 checksum은의 문제가 있다.

비용이많이드는 taint tracking 과 symbolic execution이 있다. 하지만 이러한 방법은 source code에 대한 접근, environment (OS나 library function call)에 대한 엄밀한 정보가 필요하다.

이 논문에서 taint tracking과 symbolic execution에 비해 가벼우면서 효과적인 대안을 소개한다. 이 방법은 large binary application과 알려지지 않은 environment에 쉽게 확장 가능하다. 

우리는 input의 일부분이 직접적(거의 수정되지 않은)으로 프로그램 state에 포함되는것을 관찰한다. 이러한 input-to-state 연관성은 매우 효율적인 방법으로 두가지 문제점을 극복하는데 사용된다.

우리는 이러한 방법으로 REDQUEEN을 만들었다. 주어진 바이너리 실행파일에 대해서 magic byte와 중첩된 checksum test를 자동으로 해결할 수 있다. 

REDQUEEN은 처음으로 LAVA-M에 심어진 버그의 100% 이상을 모든 타깃에서 찾아내었다. 또한 65개의 새로운 버그와 여러 프로그램 및 OS kernel에서 16개의 CVE를 얻었다.


### Grey-box concolic testing on Binary Code
[Link](https://softsec.kaist.ac.kr/~sangkilc/papers/choi-icse2019.pdf)
white-box와 grey-box fuzzing의 장점을 결합하여 새로운 path based testcase generation 방법인 grey-box concolic testing을 제시한다. 우리는 대상 프로그램의 executin path를 white-box fuzzing, 즉 concolic tesing처럼 체계적으로 탐색하는 동시해 grey-box fuzzing의 단순함을 모두 구현하였다.

이는 경량화된 계측만을 사용하여 SMT solver를 사용하지 않는다.

Eclipser라는 fuzzer를 구현하여 최신 grey-box fuzzer(AFLFast, LAF-intel, Stellix, VUzzer), smybolic executor(KLEE)와 비교하여 다른 도구들보다 더 높은 code coverage를 달성하고 더 많은 버그를 발견하였따.

## 2.  Improving the efficiency of fuzzing
### Designing New Operating Primitives to Improve Fuzzing
[Link](https://cosmoss-jigu.github.io/pages/pubs/fuzzing-xu-ccs17.pdf)

Fuzzing의 반복 실행 시간을 단축하여 fuzzing의 성능을 향상시키는 방법을 다루었다.

AFL이 120코어로 실행될때 fork() system call의 scalability로 인해 24배 느려진다는 것을 관찰하였다. 다른 fuzzer들도 비슷한 설계 패턴을 따르기 때문에 같은 scalability bottleneck이 있을것이라고 예상된다.

fuzzing 성능 향상을 위하여 이 문제를 문제를 해결하였다.

새로운 operating primitives specialized를 통하여 120코어를 사용하였고 Google fuzzer test suite를 대상으로 AFL : 6.1~28.9배, LibFuzzer : 1.1~735.7배 향상함을 확인하였다.

더 일반적인 설정인 30코어를 할당한 경우에도 AFL의 처리량을 7.7배까지 향상시킨다. 이러한 fuzzer-agnostic primitives는 어떤 fuzzer에도 쉽게 적용할 수 있으며 대규모 fuzzer 및 cloud 기반 fuzzing 서비스에 직접적인 이익이 된다.

## 3. Hybrid fuzzing
### Driller: Augmenting Fuzzing Through Selective Symbolic Execution

[Link](https://sites.cs.ucsb.edu/~vigna/publications/2016_NDSS_Driller.pdf)

### QSYM : A Practical Concolic Execution Engine Tailored for Hybrid Fuzzing

[Link](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-yun.pdf)

## 4. concolic testing

[Link](http://acm.mementodepot.org/pubs/proceedings/acmconferences_3180155/3180155/3180155.3180166/3180155.3180166.pdf)

## 5. Kernel fuzzing
### IMF: Inferred Model-based Fuzzer (MacOS)
[Link](https://acmccs.github.io/papers/p2345-hanA.pdf)
### DIFUZE: Interface Aware Fuzzing for Kernel Drivers (Linux)
[Link](https://acmccs.github.io/papers/p2123-corinaA.pdf)