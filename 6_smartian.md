[SMARTIAN: Enhancing Smart Contract Fuzzing
with Static and Dynamic Data-Flow Analyses
](https://softsec.kaist.ac.kr/~sangkilc/papers/choi-ase2021.pdf)

# 0. Abstract
smart contract는 기존 소프트웨어와 달리 일련의 transaction을 공유하는 고유한 조직이 있다. 불행히도 이러한 특성으로 인해 기존 Fuzzer가 중요한 traszction sequence를 찾는것은 어렵다. 이 문제를 해결하기 위하여 우리는 smart contract fuzzing에 대한 static, dynamic analysis를 모두 사용한다.

먼저 smart contract byte code를 static analysis를 통하여 어떤 transaction sequence가 효과적으로 테스트로 이어질지 예측하고 각 transaction이 충족해야 하는 특정 constraint가 있는지 확인한다. 이러한 정보는 fuzzing 단계로 전달되어 initial seed corpus를 구성하는데 사용된다. 

fuzzing 중에는 가벼운 dynamical data-flow analysis를 수행하여 data-flow based feedback을 통해 fuzzing을 guide한다.

우리는 *SMARTIAN*이라는 실용적인 open-source fuzzer에 이 아이디어를 구현하였다. SMARTIAN은 source code 없이 실제 smart contract에서 버그를 발견할 수 있다. 우리의 실험에서 기존의 최첨단 도구보다 더 효과적으로 CVE를 찾을 수 있었다. 또한 code coverage 측면에서 다른 도구보다 성능이 뛰어나다.

# 1. Introduction
smart contract는 수십억의 디지털 자산을 처리하는 경우가 많기에 버그가 발생하면 치명적인 영향을 미칠 수 있다. 2016 년의 DAO 공격에서 공격자는 smart contract의 reentrancy bug를 이용하여 360만 either를 훔쳤다.

당연히 smart contract 에서 bug를 자동으로 찾는것에 대해 연구 관심이 급증했지만 우리가 아는 바로는 우리가 발견한 기존 도구는 다음과 같은 문제가 존재한다.

1. coverage 측정에 필요한 test case를 생성하지 않는다. 많은 tool은 replayble test case를 생성하지 않으며 transaction에 대한 불완전한 정보를 출력한다. 또한 버그를 유발하는 test case에만 초점을 맞추고 coverage를 증가시키는 것은 무시한다. 이로 인해 tool의 적용 범위 성과를 정량적으로 비교하기 어렵다.
2. 논문은 항상 구현을 제공하거나 평가에 사용되는 dataset을 게시하지 않는다.
3. 많은 tool은 소수의 bug class에만 중점을 두어 usability에 크게 제한된다. 에를들어 *Echidna*는 assertion failure 감지, custom properties 확인만 확인할 수 있다.

이러한 관찰은 (1) replayble test case를 생서할 수 있고 (2) 공개적으로 사용 가능하며 (3) 다양한 bug class set을 찾을 수 있는 실용적인 test tool의 필요성을 시사한다. fuzzing은 이러한 요구사항을 달성하기 위한 그럴듯한 기술이지만 현재의 smart contract fuzzer중 어느것도 모두 충족하지 않는다.

이것이 유일안 요구사항은 아니다. stateful transaction을 처리하는 것은 중요한 기술적 과제이다. smart contract는 기존 application과 다르게 persistent state를 유지한다. 즉 smart contract test는 대상 계약의 persistent state를 변경하여 transaction sequence를 찾는 것이다. 안타깝게도 기존 code coverage feedback은 transaction sequence를 식별하는데 효과적이지 않을 수 있다. 

이전 fuzzer는 transaction sequence를 무작위로 변경하거나 ML을 이용하여 이 문제를 부분적으로 처리하였다. 그러나 이러한 방법들은 실폐할 가능성이 있다.

이 논문에서 EVM bytecode에 대한 static analysis와 dynamic analysis를 모두 사용하여 이 문제를 해결한다. 핵심 아이디어는 sequence의 중요성이 persistent state variable의 data dpendencie에 따라 결정될 수 있다. 따라서 persistent state variable의 dataflow를 분석하는것이 중요하다.

특히 우리는 (1) initial seed pool 생성 (2) runtime에서 seed pool 개선을 하였다.
먼저 smart contract (EVM byte code)를 static analysis를 통하여 persistent state를 수정할 수 있는 의미있는 transaction sequence를 파악한다. 이 단계에서 얻은 transaction sequence는 fuzzing에 유용한 seed가 된다. 이 과정은 전처리 단계로 한번만 수행된다.

전처리 단계에서 얻은 initial seed를 사용하여 fuzzing을 진행한다. 그러나 fuzzer는 seed pool을 효과적으로 업데이트하기 위해 runtime에 유용한 transaction sequence를 식별할 수 있어야 한다. 따라서 우리는 fuzzing동안 state variable 간의 dynamic data flow를 모니터링 하는 새로운 feedback 매커니즘인 `data-flow-based feedback`을 소개한다.

우리는 static, dynamic analysis를 통해 test중인 smart contract에 대한 중요한 transaction sequence를 체계적으로 생성할 수 있는 open-source smart contract fuzzer인 *SMARTIAN*을 설계하고 구현하였다. 우리의 기여는 다음과 같다.

1. initial seed pool generation을 위한 새로운 dynamic analysis 기술을 제안한다.
2. smart contract fuzzing을 위한 새롭고 체계적인 feedback 매커니즘인 data-flow-based feedback을 제시한다.
3. 
# 2. Background

## 2.1. Basic Terminologies
## 2.2. Smart Contract Bug Classes
## 2.3. Existing Tools
# 3. Overview
## 3.1. Motivating Example
## 3.2. Our Approach
## 3.3. Our Contribution over Previous Work
# 4. Design
## 4.1. Information Gathering
## 4.2. Seed Pool Initialization
## 4.3. Data-Flow-Based Fuzzing
# 5. Evaluation
## 5.1. Experimnetal Setup
## 5.2. Impact of Our Analysis
## 5.3. Comparioson aginst Existing Tools
## 5.4. Large-Scale Study
## 5.5. Threats to Validity

# 6. Discussion
# 7. Related Work
# 8. Conclusion
우리는 smart contract testing tool의 한계를 연구하여 여러 설계 및 기술 문제를 식별하였다.

우리는 특히 smart contract의 stateful transaction을 효과적으로 처리하는 문제를 해결 했으며 이로 인해 fuzzing중 seed를 생성하고 seed pool을 업데이트 하기 위한 static, dynamic analysis 기술을 결합하여 도입하였다. 우리의 연구에서 제안된 기법이 code coverage와 버그 발견 측면에서 효과적인 fuzzing을 가능하게 하며 무시할만한 overhead를 발생시킨다.
