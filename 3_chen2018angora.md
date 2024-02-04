[Angora: Efficient Fuzzing by Principled Search](https://web.cs.ucdavis.edu/~hchen/paper/chen2018angora.pdf)

# 0. Abstract
fuzzing은 소프트웨어 버그를 찾기 위해 널리 사용되고 있다. 그러나 최신의 fuzzer의 성능은 기대에 못미치기도 한다.

- symbolic execution 기반 : high quality input , 속도가 느림
- random muatation 기반 : low quality input, 속도가 빠름

이 논문에서는 새로운 mutation-based fuzzer인 Angora를 제시한다.

Angora의 목표는 symbolic execution 없이 path constraint를 해결함으로 branch coverage를 증가시키는 것이다. 다음과 같은 핵심 기술을 도입하였다.

- scalable byte-level taint tracking
- context-sensetive branch count
- search based on gradient descent
- input legnth exploration

LAVA-M data set에서 거의 모든 injected bug를 찾았고 다른 모든 fuzzer보다 더 많은 bug를 찾았다.

*who* 프로그램에서는 두번째로 좋은 fuzzer보다 8배 많은 버그를 찾았다. 또한 LAVA 제작자가 injected 하였지만 trigger하지 못한 103개의 버그를 찾아내었다. 

또한 8개의 널리 사용되는 open source program에서 Angora를 테스트하여 다음과 같은 버그를 발견하였다.

|program name| number of bug|
|:---:|:---:|
|file|6|
|jhead|52|
|nm|29|
|objdump|40|
|size|48|

# 1. Introduction
Fuzzer는 소프트웨어 버그를 찾는 인기있는 기술이다. Coverage-based fuzzer는 program state를 탐색하기 위한 input을 만드는 문제에 직면하였다.

## 1.1. AFL
몇몇 fuzzer는 symbolic execution을 사용하여 path constraint를 해결하지만 symbolic execution은 느리고 많은 종류의 constraint 를 효과적으로 해결할 수 없다. *AFL*은 symbolic execution과 같은 heavy weight program analysis를 사용하지 않음으로 이러한 문제를 피했다.

프로그램을 계측하여 어떤 input이 새로운 program branch를 탐색하는지를 관측한다. 그리고 이러한 input에 muatation을 주고 seed에 추가한다. AFL은 프로그램 실행에 있어서 낮은 overhead를 갖지만 AFL이 생성하는 대부분의 input은 효과가 없다. (새로운 program state를 탐색하지 못함) 왜냐하면 program의 data flow를 고려하지 않고 input을 맹목적으로 mutate하기 때문이다.

몇몇 Fuzzer들은 AFL에 heuristic을 이용하여 "magic byte"와 같은 간단한 predicates을 추가하였다.

## 1.2. Angora
우리는 symbolic execution을 사용하지 않고 path constraint를 해결하기위하여 Angora 라는 fuzzer를 설계하고 구현하였다.

Angora는 탐색되지 않은 branch를 추적하고 이러한 branch의 path constraint를 해결하려고 한다. 이를 위해 다음과 같은 기술을 도입하여 현존하는 fuzzer보다 상당히 좋은 결과를 얻었다.

### 1.2.1 Context-sensetive branch count
AFL은 context-insensitive bran count를 사용한다. 우리의 실험에 의하면 context를 branch coverage에 추가하면 Angora가 더 효과적으로 프로그램을 탐색한다.
### 1.2.2. Scalable byte-level taint tracking
많은 path constraint는 input의 몇 byte에 의존한다. 어떠한 input byte가 path constraint에 영향을 주는지 추적하여 Angora는 이러한 byte에만 mutate를 수행한다. 이러한 방식으로 탐색할 공간을 줄인다.
### 1.2.3. Search based on gradient descent
Angora는 symbolic execution을 사용하지 않고input을 path constraint를 만족하도록 mutate 한다. symbolic execution은 cost가 높고 많은 종류의 constraint를 해결하지 못한다. 

Angora는 ML분야의 gradient descent algorithm을 사용하여 path constraint를 해결한다.
### 1.2.4. Type and shape inference
input의 많은 byte는 하나의 값으로 사용 된다.
예를 들어 4 byte의 input은 32 bit signed integer로 사용된다.

검색에 gradient descent를 효율적으로 사용하기 위해서 Angora는 group의 유형을 추론한다.
### 1.2.5. Input length exploration
어떤 프로그램은 input의 길이가 특정 값을 초과할때에만 특정 state를 탐색할 수 있다. symbolic execution이나 gradient descent는 input의 길이를 증가시켜야 하는지 알려주지 않는다.

Angora는 input의 길이가 path constraint에 영향을 미칠 수 있는 경우를 감지하고 input 길이를 증가시킨다.


# 2. Background: American Fuzzy Lop (AFL)
fuzzing은 bug를 찾기 위한 자동화된 테스트 기술이다. AFL은 mutation-based graybox fuzzer로 compile-time instrumentation과 genetic algorithm을 사용하여 새로운 internal state를 실행할것으로 예상되는 test  case를 자동으로 생성한다.

AFL은 coverge-based fuzzer이고 다양한 path를 탐색하고 bug를 발생하기 위한 입력을 생성한다.
## 2.1. Branch coverage
AFL은 branch를 기반으로 path를 측정한다. AFL은 실행중 각 branch가 몇번 실행되었는지 센다. AFL은 branch를 튜플 (l_prev,l_cur)로 나타낸다. (l_prev = 이전, l_prev = 이후의 block ID)

AFL은 간단한 instrumentation을 사용하여 branch coverage정보를 얻는다. instrumentation은 compile time에 각 branch point에 삽입되고 AFL은 모든 조건문의 각 branch가 실행된 횟수를 세는 path trace table을 할당한다. table에는 분기의 hash인 h(l_prev,l_cur)이다. (h는 hash function)

AFL은 global branch coverage table을 유지한다. branch가 다른 실행에서 실행된 횟수를 기록하는 8bit vector를 포함한다. 

    b[0-7] = { [1], [2], [3], [4-7], [8-15], [16-31], [32-127], [128,∞)}

AFL은 path trace table과 branch coverage table을 비교하여 새로운 input이 새로운 internal state를 실행하는지 heuristically하게 결정한다. 다음중 하나가 발생할때 새로운 state를 실행한다

- 새로운 branch를 실행할때 : path trace table에는 있지만 branch coverage table에는 없을때
- 이 전 실행과 다른 횟수로 실행되는 branch가 있을때

## 2.2. Mutation strategies
AFL은 다음과 같은 muatation 전략을 사용한다
- bit or byte flip
- "intersting" byte or word or dword 사용
- byte or word or dword에 대한 작은 정수의 덧셈 뺄셈
- single-byte를 random 결정
- blcok 삭제, 복제(덮어쓰기, 삽입), memset
- random location에서 두 input file 이어붙이기 
# 3. Design
## 3.1. Overview
AFL과 유사한 fuzzer들은 branch coverage를 지표로 사용한다. branch coverage를 계산할때 context를 고려하지 않는다. 우리의 경험에 따르면 context 없이는 branch coverage와 program state를 충분히 탐색하지 못한다. 따라서 우리는 coverage의 지표로 context-sensitive branch coverage를 제안한다. 

### Algorithm of Angora
Algorithm 1은 Angora의 두 단계인 계측과 fuzzing loop를 보여준다. 

![Algorithm1]

fuzzing loop는 탐색되지 않은 branch를 선택하고 해당 branch를 탐색하는 input을 찾는다. input을 효율적으로 찾기 위한 기술은 다음과 같다.

1. 대부분의 조건문의 경우 조건은 input의 몇 byte에 의해서만 영향을 받는다. 즉 전체 input을 mutate하는것은 비생산적이다. 분기를 탐색할때 Angora는 해당 조건문의 input bytes flow를 결정하고 이러한 byte의 muate에만 집중한다.

2. muate할 input byte를 결정한 후 이 byte를 어떻게 mutate할지 결정한다. 무작위 또는 heuristic하게 결정한다면 알맞는 값을 효율적을 찾기 어렵다. 대신 우리는 path constraint를 입력에 대한 blackbox function의 constraint로 본다. 이를 gradient descent algorithm을 사용하여 해결한다.

3. gradient descent를 사용할때 blackbox function의 argument에 대해서 평가한다. 예르 들어 input의 연속된 4byte가 정수로 사용될때 우리는 이 4byte를 독립적인 argument가 아닌 하나의 argument로 고려해야 한다. 이 방법을 사용하기 위하여 input의 byte의 유형과 단일 또는 공동으로 사용되는지 여부를 추론해야한다.

4. 일부 버그는 input이 특정 임계값보다 길어졌을때 발생한다. 따라서 input의 byte만을 변형하는것이 아닌 길이를 변경해야 한다. 하지만 이는 딜레마를 생성한다.input이 너무 짧다면 특정 버그를 발생시키지 못할 수 있다. 반면 너무 길다면 프로그램이 느려질 수 있다. 대부분의 fuzzer는ㄴ adhoc을 사용하여 input의 길이를 변경한다. 반면 Angora는 새로운 branch를 탐색할 수 있는 더 긴 input이 필요할때를 감지하고 최소 필요 길이를 결정하는 코드로 프로그램을 계측한다.

다음은 조건문을 fuzzing하는 단계의 diagram과 코드이다.

![Figure1]

``` C
void foo(int i, int j) {
    if (i * i - j * 2 > 0) {/* some code*/}
    else{/* some code*/}
}
int main() {
    char buf[1024];
    int i = 0, j = 0;
    if(fread(buf, sizeof(char), 1024, fp)< 1024)
        return 1;
    if(fread(&i, sizeof(int), 1, fp) < 1)
        return 1;
    if(fread(&j, sizeof(int), 1, fp) < 1)
        return 1;
    foo(i, j);
}

```
다음과 같은 내용을 확인할 수 있다.
### Byte-level taint tracking
Angora는 2번째줄의 if문을 fuzzing할때 byte-level taint tracking을 사용하여 1024-1031번째 byte flow가 표현식을 결정함을 확인하여 이 byte들만 변형한다.
### Search algorithm based on gradient descent
Angora는 2번째 줄의 조건문의 두 분기를 각각 실행하는 입력을 찾아야 한다. 조건문 내 표현식을 입력 x에 대한 f(x)로 처리하고 gradient descent algorithm을 사용하여 f(x)>0, f(x')<=0인 두 input x, x'을 찾는다.
### Shape and type inference

### Input length exploration
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
