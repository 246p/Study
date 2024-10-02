# 0. fuzz.sh
``` sh
#!/bin/bash
rm -rf "$1"/fuzz
cd "$1"
cd build
rm test.bc
make clean; make
~/gllvm/get-bc test
cd ../
mkdir fuzz; cd fuzz
cp ../build/test.bc .
cp ../build/targets . ## echo 'test.c:43' >> targets
mkdir in;
echo $'a' > in/in0.txt 
~/windranger/instrument/bin/cbi --targets=targets test.bc
~/windranger/fuzz/afl-clang-fast test.ci.bc -lpng16 -lm -lz -lfreetype -o  test.ci
# llvm-dis test.bc
# llvm-dis test.ci.bc
AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1 AFL_SKIP_CPUFREQ=1 ~/windranger/fuzz/afl-fuzz -d -i in/ -o out ./test.ci
```

# 1. 오류

/windranger/instrument/src/cbi.cpp 에 `std::cout << ... `를 추가해 가며 통하여 컴파일 하여 해결

> makefile을 직접 작성해서 실험을 행하던 중 clang compiler에 `-g` 생략 >  디버깅 정보를 포함하지 않아서 ./cbi 에서 target을 식별하는 과정에서 오류가 발생하였다고 추정 

1. LLVM IR (.ll file)을 읽을줄 알아야 하는지 궁금합니다.
2. cbi.cpp에 llvm, SVF framwork가 많은데 어느정도 수준까지 알아가야하는지 궁금합니다.

# 2. DBB가 여러개 가능한 이유
- cbi.cpp의 output중 `function.txt` 가 potential DBB 역할, `distance.txt`에 target으로의 거리가 있음
- afl-fuzz.c의 critical_ids 변수가 potential DBB를 의미함


# 3. effector map을 생성하는 branch
- afl-fuzz.c:fuzz_one:7346에서 update 

``` c
if (!eff_map[EFF_APOS(stage_cur)]) {
    u32 cksum;
    /* If in dumb mode or if the file is very short, just flag everything
    without wasting time on checksums. */
    if (!dumb_mode && len >= EFF_MIN_LEN)
        cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);
    else
        cksum = ~queue_cur->exec_cksum;
    if (cksum != queue_cur->exec_cksum) {
        eff_map[EFF_APOS(stage_cur)] = 1;
        eff_cnt++;
    }
}
```

# 4. Simulated Annealing 
afl-fuzz.c:calculate_score:6248

```c
double power_factor = 1.0;
if (!afl_flag) {
  u8 flag = 1;
  if (!no_critical_flag && explore_status)
  flag = 0;
    
  if (flag) {
    if (q->distance > 0) {
      double normalized_d = 0; // when "max_distance == min_distance", we set the normalized_d to 0 so that we can sufficiently explore those testcases whose distance >= 0.
      if (max_distance != min_distance)
        normalized_d = (q->distance - min_distance) / (max_distance - min_distance);
        if (normalized_d >= 0) {
          double p = (1.0 - normalized_d) * (1.0 - T) + 0.5 * T;
          power_factor = pow(2.0, 2.0 * (double) log2(MAX_FACTOR) * (p - 0.5));
        }// else WARNF ("Normalized distance negative: %f", normalized_d);
      }
    }
  }
  perf_score *= power_factor;
```

- explore_status = 1 인 경우 (explore stage)에서는 power_factor가 simulated Annealing을 통하여 감소
- exploite stage의 경우 power_factor가 변경되지 않는다. 

# 5. seed priorization

main:10668 > `explore_status`에 따라 `cb_cull_queue` 또는 `cull_queue`가 실행됨

- cb_cull_queue : expoloite stage
- cull_queue : explore stage : AFL과 동일함

# 6. fuzzer test set 선정 기준
## UniBench RQ1, RQ3
- 20 real world program, (6 categories) 
- 4 target site per program
- table 1는 해당 실험중 일부 [Link](https://sites.google.com/view/windranger-directed-fuzzing/time-to-target-results?authuser=0)
- 
## AFLGo RQ2
- Test suite

## Google Fuzzer Test Suite RQ2
- Table3

# 7. experience

`test.c`

``` c
#include <stdio.h>
#define BUG() 	int *a=NULL; \
				        *a=1;
#define LEFT(a) (a>='A') ? 1 : 0
#define RIGHT(a) (a<'A') ? 1 : 0
char buf[10];
void fun0(void);
void fun1(void);
void fun2(void);
void fun3(void);
void fun4(void);
void fun5(void);
void fun6(void);
void fun7(void);
void fun8(void);
void fun9(void);
int main(){ // 0
	fgets(buf,10,stdin);
	fun0();
	return 0;
}
void fun0(void){ // 1
	if(LEFT(buf[0])) fun1();
	if(RIGHT(buf[0])) fun2();
}
void fun1(void){ // 2
	if(LEFT(buf[1])) fun3();
	if(RIGHT(buf[1])) fun8();
}
void fun2(void){ //3 
	if(RIGHT(buf[1])) fun4();
}
void fun3(void){ //8
	BUG()
}
void fun4(void){ //5
	if(LEFT(buf[2])) fun5();
	if(RIGHT(buf[2])) fun6();
}
void fun5(void){//5
	if(LEFT(buf[3])) fun7();
	if(RIGHT(buf[3])) fun9();
}
void fun6(void){//6
	BUG()
}
void fun7(void){//4
	BUG()
}
void fun8(void){
	printf("8");return ;
}
void fun9(void){
	printf("9");return ;
} 
```

`function.txt`
``` txt
0 in line: 17 file: test.c
1 in line: 22 file: test.c
2 in line: 26 file: test.c
3 in line: 30 file: test.c
4 in line: 36 file: test.c
5 in line: 40 file: test.c
6 in line: 44 file: test.c
7 in line: 47 file: test.c
8 in line: 33 file: test.c
```

`distance.txt`
``` txt
0 255 0 { ln: 18  cl: 15  fl: test.c }
1 331 0 { ln: 23  cl: 5  fl: test.c }
2 281 0 { ln: 23  cl: 5  fl: test.c }
3 281 0 { ln: 23  cl: 5  fl: test.c }
4 400 0 { ln: 23  cl: 19  fl: test.c }
5 542 0 { ln: 24  cl: 5  fl: test.c }
6 442 1 { ln: 24  cl: 5  fl: test.c }
7 442 1 { ln: 24  cl: 5  fl: test.c }
8 342 0 { ln: 24  cl: 20  fl: test.c }
9 400 0 { ln: 27  cl: 5  fl: test.c }
10 300 1 { ln: 27  cl: 5  fl: test.c }
11 300 1 { ln: 27  cl: 5  fl: test.c }
12 200 0 { ln: 27  cl: 19  fl: test.c }
13 440 0 { ln: 31  cl: 5  fl: test.c }
14 340 1 { ln: 31  cl: 5  fl: test.c }
15 340 1 { ln: 31  cl: 5  fl: test.c }
16 240 0 { ln: 31  cl: 20  fl: test.c }
17 300 0 { ln: 37  cl: 5  fl: test.c }
18 250 0 { ln: 37  cl: 5  fl: test.c }
19 250 0 { ln: 37  cl: 5  fl: test.c }
20 400 0 { ln: 37  cl: 19  fl: test.c }
21 400 0 { ln: 38  cl: 5  fl: test.c }
22 300 1 { ln: 38  cl: 5  fl: test.c }
23 300 1 { ln: 38  cl: 5  fl: test.c }
24 200 0 { ln: 38  cl: 20  fl: test.c }
25 400 0 { ln: 41  cl: 5  fl: test.c }
26 300 1 { ln: 41  cl: 5  fl: test.c }
27 300 1 { ln: 41  cl: 5  fl: test.c }
28 200 0 { ln: 41  cl: 19  fl: test.c }
29 0 0 { ln: 45 fl: test.c }
30 0 0 { ln: 48 fl: test.c }
31 0 0 { ln: 34 fl: test.c }
```
