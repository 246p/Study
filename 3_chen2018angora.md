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
f(x)는 vector x에 대한 함수이다. gradient descent를 적용할때 Angora는 x의 각 component에 대해 편미분을 계산해야 하므로 각 component와 유형을 결정해야한다.
### Input length exploration
input이 1032byte보다 작을때 mian은 foo를 호출하지 않는다. 무작정 긴 input을 시도하는 대신 input에서 읽는 일반적인 함수들을 계측하고 더 긴 입력이 새로우 state를 탐색할 지 결정한다.

## 3.2. Context-sensitive branch count
### context-insensetive
위에서 AFL의 branch coverage table에 대해서 설명한다. 이러한 설계는 다음과 같은 장점이 있다.

1. 공간을 효율적으로 사용한다. branch의 수는 프로그램의 크기에 선형적이다.
2. range를 사용하여 branch execution count를 사용하면 다음 실행이 새로운 내부 상태를 나타내는지에 대한 heuristic을 제공한다.

하지만 이러한 설계는 한계가 있다. AFL의 branch는 context-insensetive이기 때문에 같은 branch의 다른 context에서의 실행을 구분하지 못하며 프로그램의 내부 새로운 내부 상태를 넘어갈 수 있다. 다음은 이에 대한 예시이다.
```C
void f(bool x){
    static bool trigger = false;
    if (x)
        if(trigger)
            if(input[2])
                // crash

    else 
        if (!trigger)
            trigger = true;


}

bool [] input;

int main(){
    f(input[0]);
    ...
    f(input[1]);
}
```

`if(x)`의 branch coverage를 고려해 보자.
첫번째 입력이 `10`이라면 19번째 줄에서 `f()`를 호출할때 true branch를 실행히킨다. 이후 21줄에서 `f()`를 실행시킬때 10번째줄의 false를 실행시킨다.

AFL의 branch가 context-insensetive 이기에 두 input 모두 실행되었다고 생각한다. 

그 이후 `01`이 입력으로 들어온다면 AFL은 이전 입력에서도 역시 3,10 branch 모두 실행되었기 때문에 이 입력은 새로운 내부 state를 trigger하지 않는다고 생각한다. 하지만 이 입력은 새로운 내부 상태를 trigger 한다. input[2]==1일때 crash를 발생시킨다.

※ 논문에서 오타가 있는것 같다. 코드와 설명 모두 input[2]가 아닌 input[1]로 해석해야 할것 같다.
### context-sensetive
우리는 branch를 tuple (l_prev, l_cur, context)로 정의하여 context를 포함한다.

l_prev와 l_cur는 조건문 전 후의 baisc block의 ID 이고 context는 h(stack)이다. 예를들어 위 코드를 `10`으로 실행할때 19줄의 `f()`로 들어간 이후에는 branch (l_3, l_4, [l19])를 실행한다. 이후 21줄의 `f()`를 실행한 경우에는 branch (l_3, l_10, [l21])을 실행한다.
반면 `01`의 input으로 실행될 때에는 (l_3, l_10, [l19])를 실행한 후 (l_3, l_4, [l21])을 실행하기 때문에 barnch의 정의에 context를 포함시킴으로서 Angora는 두번째 input이 새로운 내부 state를 trigger한다는것을 감지할 수 있다.

### using hash function
branch에 context를 추가한다면 unique branch의 수가 증가하며 deep recursion이 발생할 수 있다. 현재 구현은 call stack의 hash를 계산하기 위하여 hash function h를 사용함으로 이 문제를 완화한다.

h는 stack상의 모든 call site의 ID의 xor을 계산한다. Angora가 프로그램을 계측할때 각 call site에 랜덤 ID를 할당한다. 함수가 f가 자신을 재귀적으로 호출할때 Angora가 동일한 call site의 ID를 call stack 얼마나 많이 push하든 h(stack)은 최대 2개의 고유값을 출력한다.

즉 함수 f 내의 unique branch의 수를 최대 2배까지만 증가시킨다. 실제 프로그램에 대한 평가는 context를 포함한 후 unique branch가 최대 7.21배 까지 증가한다는 것을 보여준다. 이는 code coverage가 증가한다는 이점을 얻기 위한 대가이다.
## 3.3. Byte-level taint tracking
### taint tracking
Angora는 탐색되지 않은 branch를 실행하는 입력을 생성하는것을 목표로 한다. 탐색되지 않은 branch를 실행하려고 할때 그 branch의 predicate에 따라 영향을 미치는 input내의 byte offset을 알아내야한다. 이를 위하여 byte-level taint analysis를 사용한다.

taint analysis는 비용이 많이 든다. 특히 개별 byte를 추적할때는 더욱 그렇다. 이로 인하여 AFL은 이를 사용하지 않는다.

우리는 대부분의 프로그램 실행에서 taint tracking이 불필요하다고 생각한다. input에 대하여 한번 taint tracking을 실행한다면 각 조건문으로 흐르는 byte offset을 기록할 수 있다. 그 이후 byte를 mutate할때 taint tracking을 하지 않을 수 있다.

이는 taint tarcking의 비용을 많은 mutation에 걸쳐 분산시킨다. 이로 인하여 Angora는 AFL과 유사한 양의 input execution을 처리할 수 있다.

### data structure of taint label
Angora는 프로그램내 각 변수 x를 taint label t_x와 연결한다. taint label의 data구조는 메모리사용량에 큰 영향을 미친다.

단군한 구현 방법은 각 taint label을 bit vector로 표현하는 것이며 여기서 각 비트 i는 input의 i번째 byte를 나타낸다. 하지만 bit vector의 크기가 input의 크기에 선형적으로 증가하기 때문에 이러한 data구조는 큰 입력에 대해서 사용하기 힘들다.

taint label의 크기를 줄이기 위하여 bit vetor를 table에 저장하고 table의 index를 taint label로 사용할 수 있다. table 항목중 logarithm이 가장 긴 vector의 길이보다 작은 경우가 있는데 이를 이용하여 taint label의 크기를 크게 줄일 수 있다.

#### using table
하지만 다음과 같은 연산을 지원해야 한다.
- INSERT(b) : bit vector b를 삽입하고 그 라벨을 반환한다.
- FIND(t) : taint label t에 대한 bit vector를 반환한다.
- UNION(tx,ty) : t_x, t_y의 합집합을 표현하는 taint label을 반환한다.

FIND는 저렴하지만 Union은은 비용이 많이든다. UNION은 다음과 같은 단계를 거친다.
1. 두 label의 bit vector를 찾아 합집합 u를 계산한다.
2. u가 이미 존재하는지 여부를 결정하기 위하여 table을 검색한다. 존재하지 않는다면 u를 추가한다. 이때 선형 검색을 하는데 많은 비용이 든다.
   
이를 해결하기위하여 hash set을 구축할 수 있겠지만 많은 bit vector가 존재하고 각 bit vector가 길다면 hash code를 계산하는데 많은 시간이 걸리고 hash set을 저장하는데 많은 공간이 필요하다.

산술 표현힉에서 taint data를 추적할때 UNION이 흔한 연산이므로 효율적이여야 한다. 서로 다른 vector가 같은 위치에 1을 가질 수 있기 때문에 UNION-FIND 구조를 사용할 수 없다.

#### using binary tree
bit vector를 저장하기 위한 data structure를 도입한다. 이 data structure는 효율적인 INSERT, FIND, UNION을 가능하게 한다. 각 bit vector는 부호 없는 정수를 사용하여 고유한 라벨을 할당한다. 새 bit vector를 삽입할때 다음으로 사용가능한 부호없는 정수를 할당한다. 이를 위하여 다음 두가지 요소를 사용해야한다.

- binary tree를 사용하여 bit vector를 label로 mapping한다.
- look up table은 label을 bit vector로 mapping한다.
  
이 데이터 구조를 사용하여 bit vector를 효율적으로 관리하고 label을 사용하여 bit vector를 빠르게 찾아내며 UNION연산을 수행할때 필요한 bit vector의 합집합을 효과적으로 결정할 수 있다. 메모리 사용량을 최소화 하며 필요한 연산을 신속하게 수행할 수 있는 방법을 제공한다.

이 data structure에서 tree의 모든 leaf는 bit vector를 대표한다. 내부 node는 bit vector를 표현하지 않는다. 그래서 tree 에는 불필요한 많은 node를 가 있다. 

예시를 들기 전에 regular expression에 대해 알아본다. 먼저 `x*` 는 x가 0개 이상 반복된다는 의미이다. 또한 `[xy]`는 x또는 y를 의미한다.

1. `x00*`가 tree에 있다.
2. `x0[01]*1[01]*`가 tree에 없다.

이때 x를 대표하는 어떤 node도 저장할 필요 없다.
x는 오직 하나의 decedent를 가지며 그 decedent는 leaf이기 때문이다. 그리고 그것은 `x00*`로 표현된다. 따라서 우리는 tree에 vector를 삽입할때 다음과 같이 정리 할 수 있다.

1. 벡터 끝의 모든 0을 제거한다.
2. 벡터의 첫 bit부터 마지막 bit까지 bit를 따라 tree를 순회한다. child가 없으면 생성한다.
   - bit가 0이라면 left child
   - 그렇지 않으면 right child
3. 마지막으로 방문한 node에 vector의 label을 저장한다.
이 절차를 통하여 bit vector를 저장할때 필요없는 node의 생성을 방지하고 data structure의 효율성을 높일 수 있다. vector가 많고 긴 경우에도 효율적이다.

Algorithm 2는 이 삽입 연산을 자세히 설명한다. Algorithm 3은 FIND, 4는 UNION연산을 설명한다. node를 생성할때 초기에는 label이 없음을 고려해야 한다. bit vector를 삽입할때 마지막으로 방문한 노드가 label이 없는 노드인 경우 우리는 이 node에 bit vecotr의 label을 저장한다. 이 tree는 다음 특성을 갖는다.

- leaf node는 label을 저장한다.
- 내부 node는 label을 포함할 수 있지만 내부 node의 label은 교체하지 않는다.

![Algorithm2]()

![Algorithm3]()

![Algorithm4]()

이 구조를 이용하여 bit vecotr를 저장하는데 필요한 메모리 사용량을 크게 줄인다.

## 3.4. Search algorithm based on gradient descent
byte-level taint tracking은 어떤 byte offset이 조건문으로 flow되는지 발견할 수 있다. input을 matate하여 탐색되지 않은 branch를 탐색한다. 대부분의 fuzzer는 무작 또는 조잡한 heuristic을 사용하지만 빠르게 적절한 input을 찾기 힘들다. 반면 이 논문에서는 이를 serach problem으로 보고 machine learning에서의 serach algorithm을 사용하였다. 여기서는 gradient descent를 사용하였지만 다른 search algorithm을 사용할 수 있다.

### transform comparison into constraint
이 접근 방식에서 branch를 실행하기 위한 predicate를 blackbox funtion *f(x)* 에 대한 constraint로 본다. x는 predicate로 흐르는 input value들의 vector이고 *f()*는 프로그램 시작부터 이 predicate까지의 path에서의 계산을 감지한다. *f(x)*에는 3가지 constraint가 있다.

1. *f(x)*<0
2. *f(x)<=0
3. *f(x)*=0

다음 표는 모든 형태의 비료를 3가지 유형의 constraint로 변환할 수 있음을 보여준다. 조거문의 조건에 논리연산자 `&&`또는 `||`이 포함된 경우 Angora는 이 조건문을 여러 조건문으로 분할한다. 예를들어
`if (a && b) {s} else {t}` 를 `if (a) { if (b) {s} else {t} } else {t}`로 분할한다.
|Comparison|*f*|Constraint|
|:---|:---|:---|
|a<b|*f*=a-b|*f*<0|
|a<=b|*f*=a-b|*f*<=0|
|a>b|*f*=b-a|*f*<0|
|a>=b|*f*=b-a|*f*<=0|
|a==b|*f*=abs(a-b)|*f*==0|
|a!=b|*f*=-abs(a-b)|*f*<=0|

### Gradient decent
algorithm 5는 serach algorithm을 보여준다. 초기 x_0에서 시작하여 *f(x)*가 constraint를 만족하는 x를 찾는다. 각 유형의 constraint를 만족하기위해서 *f(x)*를 최소화 해야 하며 이때 gradient descent를 사용한다. 

gradient descent는 반복적으로 작용하여 x에서 시작하여 *f(x)*의 기울기$∇_xf(x)$ 를 계산하고 x를 $x-ϵ∇_xf(x)$ 로 업데이트 한다. 여기서 $ϵ$는 learning rate이다.

![algorithm5]()

신경망을 훈련할때 연구자들은 훈련 오류를 최소화하는 가중치 집합을 찾기 위하여 gradient descent를 사용한다. 하지만 이 방법은 때때로 local minimum에 갇힐 수 있고 이는 global minimum이 아닐 수 있다. 다행이 fuzzing에서는 이는 문제가 되지 않는다. 우리는 global에서도 최적화 된 x 대신 적당히 좋은 x를 찾으면 되기 때문이다. 예를들어 constraint가 *f(x)*<0 라면 f(x)가 global minimum이 아닌 *f(x)*<0을 만족하는 x를 찾으면 되기 때문이다.  

### Problem of Gradient decent
fuzzing에 gradient descent를 사용할때는 새로운 문제점이 발생한다. 
1. $∇_xf(x)$ 를 계산하는 것이다. 신경망 에서는 $∇_xf(x)$ 를 analytic form으로 작성할 수 있지만 fuzzing에서는 *f(x)* 의 analytic form이 없다.

2. 신경망에서 *f(x)* 는 연속 함수이지만 fuzzing 에서는 보통 이산함수이다. 대부분의 변수들이 이산적이기 때문에 x의 대부분의 요소가 이산적이기 때문이다.

### numerical approximation of gradient decent
이 문제를 numerical approximation을 이용하여 해결하였다. *f(x)* 의 gradient는 각 점에 x에서 어떤 단위 벡터 v와의 내적이 v방향으로의 *f(x)*의 directional diveration인 고유한 vector field 이다. 우리는 directional diveration을 다음과 같이 근사하였다.

$∂f(x)\over∂x_i$ $=$ $f(x+δv_i)-f(x)\overδ$ 

여기서 δ는 작은값 (ex 1) 이고 $v_i$는 i번째 차원의 단위벡터이다.

각 directional diveration을 계산하기 위하여 우리는 프로그램을 두번 실행해야 한다. 한번은 $x$로 두번째는 $x+δv_i$로 실행한다.

두번째 실행에서 $f(x+δv_i)$가 계산되는 프로그램의 지점에 도달하지 못할 수 있다. 이런 상황이 발생하면 우리는 $δ$를 작은 음수 값 (ex -1)로 설정하고  $f(x+δv_i)$를 다시 계산해보려고 한다. 이것이 성공한다면 directional diveration을 계산할 수 있다. 그렇지 않는다면 우리는 diveration을 0으로 설정하여 gradient decent가 x를 변경하지 못하도록 한다. 이 계산방법은 x의 길이에 비례한 시간이 걸린다.

이론적으로 gradient decent는 모든 constraint를 해결할 수 있다. 하지만 얼마나 빠르게 해결할 수 있는지는 mathmatical funtion의 복잡성에 달려있다.

- 만약 $f(x)$가 단조롭거나 볼록하다면 $f(x)$가 복잡한 analytic form을 가지고 있더라도 해결책을 빠르게 찾을 수 있다.
- gradient decent이 찾는 local minimum이 constraint를 만족한다면 solution을 찾는것도 빠르다.
- 만약 local minimum이 constraint를 만족하지 않는다면 Angora는 다른값 x'으로 무작위 이동하여 거기서 다시 gradient decent를 수행하여 constraint를 만족하는 다른 local minimum을 찾기를 희망한다.

Angora는 $f(x)$의 analytic form을 생성하지 않고 오히려 프로그램을 실행하며 $f(x)$를 계산한다.
## 3.5. Shape and type inference
단순히 x의 각 요소를 조건문으로 흐르는 input의 byte로 설정할 수 있다. 하지만 이는 type mismatch로 인하여 gradient decent에서 문제를 일으킬 수 있다. 예를들어 input의 연속된 4 byte $b_3b_2b_1b_0$을 정수로 처리하고 $x_i$가 이 값응ㄹ 대표하도록 하자. 우리는 $f(x+δv_i)$를 계산할때 δ를 더해야 한다. 

하지만 만약 각 byte에 대하여 $x_i$로 단수히 할당한다면 각 byte에 대하여 $f(x+δv_i)$를 계산하게된다. 프로그램에서는 byte들을 단일 값으로 결합하고 표현식에서는 결합된 값을 사용하기에 LSB를 제외한 byte에 대해서 δ를 더하는 것은 값을 크게 변경하게 되며 편미분의 의미에서 벗어난다.

이 문제를 피하기위하여 (1)프로그램에서 항상 단일 값으로 함께 사용되는 input의 byte들과 (2)그 유형에 대하여 추론해야한다.

(1)의 문제를 shape inference, (2)의 문제를 type inference 라고 한다. 이는 dynamic taint analysis중에 해결된다.

### shape inference
shape inference를 위하여 초기에 모든 입력을 독립적이라고 본다. taint analysis 동안 input byte를 sequence로 읽을때 Angora는 이러한 byte들을 같은 value에 포함되는것으로 tag한다. 만약 이 과정속에서 겹치는 것이 있다면 더 작은 크기를 선택한다.

### type inference
type inference를 위하여 Angora는 값에 작용하는 명령어의 semantic에 의존한다. 예를들어 명령어가 signed integer에 작용한다면 Angora는 해당 피연산자를 signed integer로 추론한다. 같은 값이 signed, unsigned로 모두 사용될 경우 Angora는 unsigned로 처리한다. 만약 이러한 type inference가 실패하더라도 gradient descent를 이용하여 solution을 찾을 수 있다. 시간이 오래 걸릴 뿐이다.
## 3.6. Input length exploration
Angora는 다른 fuzzer에 비해 더 작은 input으로 fuzzing을 시작한다. 그러나 일부 branch는 input length가 특정 값보다 클때 실행된다.

fuzzer가 너무 짧은 input을 사용한다면 그 분기를 탐색할 수 없다. 하지만 너무 긴 input을 사용한다면 프로그램이 느려지고 메모리 부족으로 실행이 단될 수 있다. 대부분의 fuzzer는 adhoc 방식을 사용하여 다양한 입력을 시도한다. 반면 Angora는 새로운 branch를 탐색할 수 있을때만 input의 길이를 증가시킨다.

tacking을 수행하는 동안 Angora는 read-like function을 호출할대 destination memory를 input byte offset과 연결한다. 또한 read call에서 반환 값에 특별한 label을 표시한다.

반환 값이 조건문에서 사용되고 constraint를 만족하지 않는 경우 Angora는 read call이 요쳥하는 모든 byte를 얻을 수 있도록 input length를 증가시킨다. 이러한 기준이 모든 경우를 포함하는것은 아니지만 프로그램이 우리가 예상하지 못한 방식으로 input을 사용하고 그 길이를 확인할 수 있지만 그것을 발견하면 Angora에 기준을 추가하는것은 쉽다.
# 4. Implementation
## 4.1. Instrumentation
Angora는 LLVM pass를 사용하여 프로그램을 계측함으로써 실행 파일을 생성한다. 계측은 다음 과정을 수행한다.

1. 조건문의 기본 정보를 수집하고 taint analysis를 통하여 input byte offset과 연결한다. Angora는 각 입력에 대해 이 단계를 한번만 실행한다.
2. 새로운 ipnt을 식별하기위하여 excution trace를 기록한다.
3. runtime에 context를 사용한다.
4. 조건문에서 표현된 값을 수집한다.

이러한 계측 과정은 프로그램의 실행동안 필요한 데이터를 수집하여 Angora가 더 효과적으로  fuzzing을 수행할 수 있도록 한다.

Angora는 byte-level taint tracking을 위하여 *DataFlowSanitizer (DFSan)*을 확장하여 구현하였다. FIND와 UNION 기능에도 caching 기능을 추가하여 속도를 향상시켰다.

Angora는 LLVM 4.0.0 (DFSan 포함)에 기반하였다. LLVM pass은 DFSan을 제외하고 C++ 820줄 코드로 이루어져 있으며 runtime에는 taint label을 저장하기위한 data structure, input을 오염시키고 조건문을 추적하기 위한 hook을 포함하여 C++ 1950로 구현하였다.

두개의 분기를 가진 if문 외에도 *LLVM IR*은 여러 branch를 도입할 수 있는 switch문도 지원한다. Angora는 switch 문을 if문을 변환한다.

Angora는 조건문에서 문자열과 배열을 비교하는 libc function을 인식한다 예를들어 `strcmp(x,y)`를 `x strcmp y`로 변환한다 여기서 strcmp는 Angora가 이해하는 특별한 비교연산이다.



## 4.2. Fuzzer
Angora를 4488줄의 Rust코드로 구현하였다. fork server, CPU binding과 같은 기술로 최적화 하였다.
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
