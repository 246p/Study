# 1. calculate_disatcne
`windragner-fuzz:919 == aflgo-fuzz:948`  


# 2. simulated annealing
`windranger-fuzz:6248 == aflgo-fuzz:4953`

# 3. calculated_cb_distance
critical_bits[i] == 1 or 2 가 의미하는 바

`wnidranger-fuzz:setup_shm:1830` 에서 critical_bits에 대한 shared meemory 하당

``` c
s32 shm_id2 = shmget(IPC_PRIVATE, MAP_SIZE, IPC_CREAT | IPC_EXCL | 0600);
if (shm_id2 < 0) PFATAL("shmget() failed");

...

critical_bits = shmat(shm_id2, NULL, 0);
if (!critical_bits) PFATAL("shmat() failed");
```

# 4. sniff_mask


# 5. intrument

``` assembly
0000000000400f80 <fun2>:
// 기존 코드
  400f80:	55                   	push   %rbp
  400f81:	48 89 e5             	mov    %rsp,%rbp

// afl 도입
  400f84:	48 c7 c6 fc ff ff ff 	mov    $0xfffffffffffffffc,%rsi  //
  400f8b:	64 48 63 0e          	movslq %fs:(%rsi),%rcx
  400f8f:	48 8b 15 22 11 20 00 	mov    0x201122(%rip),%rdx        # 6020b8 <__afl_area_ptr>
  400f96:	48 81 f1 97 58 00 00 	xor    $0x5897,%rcx
  400f9d:	8a 04 0a             	mov    (%rdx,%rcx,1),%al
  400fa0:	04 01                	add    $0x1,%al
  400fa2:	88 04 0a             	mov    %al,(%rdx,%rcx,1)
  400fa5:	64 c7 06 4b 2c 00 00 	movl   $0x2c4b,%fs:(%rsi)

// windranger 도입 > target에 대해서만 수행되는 부분
  400fac:	48 8b 04 25 b8 20 60 	mov    0x6020b8,%rax
  400fb3:	00 
  400fb4:	48 8b 88 00 00 01 00 	mov    0x10000(%rax),%rcx
  400fbb:	48 81 c1 00 00 00 00 	add    $0x0,%rcx
  400fc2:	48 89 88 00 00 01 00 	mov    %rcx,0x10000(%rax)
  400fc9:	48 8b 88 08 00 01 00 	mov    0x10008(%rax),%rcx
  400fd0:	48 81 c1 01 00 00 00 	add    $0x1,%rcx
  400fd7:	48 89 88 08 00 01 00 	mov    %rcx,0x10008(%rax)
  400fde:	c6 80 10 00 01 00 01 	movb   $0x1,0x10010(%rax)


//기존 코드
  400fe5:	48 c7 45 f8 00 00 00 	movq   $0x0,-0x8(%rbp)
  400fec:	00 
  400fed:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400ff1:	c7 00 01 00 00 00    	movl   $0x1,(%rax)
  400ff7:	5d                   	pop    %rbp
  400ff8:	c3                   	retq   
  400ff9:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
```


`afl-llvm-pass.so.cc:143`

``` cpp
    GlobalVariable *AFLMapPtr = (GlobalVariable*)M.getOrInsertGlobal("__afl_area_ptr",PointerType::get(Int8Ty, 0),[]() -> GlobalVariable* {
        return new GlobalVariable(*M2, PointerType::get(IntegerType::getInt8Ty(M2->getContext()), 0), false, GlobalValue::ExternalLinkage, 0, "__afl_area_ptr");});

    ...

    LoadInst *MapPtr = IRB.CreateLoad(AFLMapPtr);
    MapPtr->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
    Value *MapPtrIdx = IRB.CreateGEP(MapPtr, IRB.CreateXor(PrevLocCasted, CurLoc));
```