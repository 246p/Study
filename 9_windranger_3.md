# 1. calculate_disatcne
`windragner-fuzz:919 == aflgo-fuzz:948`  


# 2. simulated annealing
`windranger-fuzz:6248 == aflgo-fuzz:4953`



# 3. intrument

``` assembly
0000000000400f80 <fun2>:
; 기존 코드
  400f80:	55                   	push   %rbp
  400f81:	48 89 e5             	mov    %rsp,%rbp

; afl이 도입
  400f84:	48 c7 c6 fc ff ff ff 	mov    $0xfffffffffffffffc,%rsi  //
  400f8b:	64 48 63 0e          	movslq %fs:(%rsi),%rcx
  400f8f:	48 8b 15 22 11 20 00 	mov    0x201122(%rip),%rdx        # 6020b8 <__afl_area_ptr>
  400f96:	48 81 f1 97 58 00 00 	xor    $0x5897,%rcx
  400f9d:	8a 04 0a             	mov    (%rdx,%rcx,1),%al
  400fa0:	04 01                	add    $0x1,%al
  400fa2:	88 04 0a             	mov    %al,(%rdx,%rcx,1)
  400fa5:	64 c7 06 4b 2c 00 00 	movl   $0x2c4b,%fs:(%rsi)

; windranger 도입 (pDBB에 대해서만 도입)
  400fac:	48 8b 04 25 b8 20 60 	mov    0x6020b8,%rax
  400fb3:	00 
  400fb4:	48 8b 88 00 00 01 00 	mov    0x10000(%rax),%rcx
  400fbb:	48 81 c1 00 00 00 00 	add    $0x0,%rcx
  400fc2:	48 89 88 00 00 01 00 	mov    %rcx,0x10000(%rax)
  400fc9:	48 8b 88 08 00 01 00 	mov    0x10008(%rax),%rcx
  400fd0:	48 81 c1 01 00 00 00 	add    $0x1,%rcx
  400fd7:	48 89 88 08 00 01 00 	mov    %rcx,0x10008(%rax)
  400fde:	c6 80 10 00 01 00 01 	movb   $0x1,0x10010(%rax)


; 기존 코드
  400fe5:	48 c7 45 f8 00 00 00 	movq   $0x0,-0x8(%rbp)
  400fec:	00 
  400fed:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400ff1:	c7 00 01 00 00 00    	movl   $0x1,(%rax)
  400ff7:	5d                   	pop    %rbp
  400ff8:	c3                   	retq   
  400ff9:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
```

`afl-llvm-pass.so.cc`

``` cpp
    // __afl_area_ptr 참조, 없다면 생성
    GlobalVariable *AFLMapPtr = (GlobalVariable*)M.getOrInsertGlobal("__afl_area_ptr",PointerType::get(Int8Ty, 0),[]() -> GlobalVariable* {
        return new GlobalVariable(*M2, PointerType::get(IntegerType::getInt8Ty(M2->getContext()), 0), false, GlobalValue::ExternalLinkage, 0, "__afl_area_ptr");
        });

    GlobalVariable *AFLPrevLoc = new GlobalVariable(M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_loc", 0, GlobalVariable::GeneralDynamicTLSModel, 0, false);

    ...

    // instruction을 삽입할 지점을 찾음
    BasicBlock::iterator IP = BB.getFirstInsertionPt();
    IRBuilder<> IRB(&(*IP));

    if (AFL_R(100) >= inst_ratio) continue;

      /* Make up cur_loc */

    // shared meemory내에 고유한 값을 생성함
    unsigned int cur_loc = AFL_R(MAP_SIZE);

    ConstantInt *CurLoc = ConstantInt::get(Int32Ty, cur_loc);

      /* Load prev_loc */

    // AFLPrevLoc에서 PrevLoc 로드
    LoadInst *PrevLoc = IRB.CreateLoad(AFLPrevLoc);
    // sanitizing을 방지하는 코드 삽입
    PrevLoc->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
    
    // prev_loc에 zero extension을 수행 (XOR 연산을 위함)
    Value *PrevLocCasted = IRB.CreateZExt(PrevLoc, IRB.getInt32Ty());
    
      /* Load SHM pointer */
    
    // AFLMap의 주소 값 로드
    LoadInst *MapPtr = IRB.CreateLoad(AFLMapPtr);
    MapPtr->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
    // MaPPtrIdx 생성 : XOR연산을 통하여 고유한 인덱스를 생성함
    Value *MapPtrIdx = IRB.CreateGEP(MapPtr, IRB.CreateXor(PrevLocCasted, CurLoc));
    
    /* Update bitmap */
    // Counter역할을 하는 명령어 생성 
    LoadInst *Counter = IRB.CreateLoad(MapPtrIdx);
    Counter->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));
    // 카운터 값을 1 증가시킴
    Value *Incr = IRB.CreateAdd(Counter, ConstantInt::get(Int8Ty, 1));
    IRB.CreateStore(Incr, MapPtrIdx)->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));

    /* Set prev_loc to cur_loc >> 1 */
    // 다음에 사용할 AFLPrevLoc을 업데이트
    StoreInst *Store = IRB.CreateStore(ConstantInt::get(Int32Ty, cur_loc >> 1), AFLPrevLoc);
    Store->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(C, None));


```

`cbi.cpp`

``` cpp

    uint32_t distance = (uint32_t)(100 * dTb[bb]);

    ConstantInt *Distance = ConstantInt::get(LargestType, (unsigned) distance);
    // instruction을 삽입할 지점을 찾음
    BasicBlock::iterator IP = bb->getFirstInsertionPt();
    llvm::IRBuilder<> IRB(&(*IP));

    /* Load SHM pointer */
    // AFLMap의 주소값 로드하여
    LoadInst *MapPtr = IRB.CreateLoad(AFLMapPtr);
    MapPtr->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

    /* Add distance to shm[MAPSIZE] */
    // distance를 저장할 포인터 생성
    Value *MapDistPtr = IRB.CreateBitCast(IRB.CreateGEP(MapPtr, MapDistLoc), LargestType->getPointerTo());

    // MapDist 의 값을 로드
    LoadInst *MapDist = IRB.CreateLoad(MapDistPtr);
    MapDist->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

    // 로드한 값에 거리를 더함
    Value *IncrDist = IRB.CreateAdd(MapDist, Distance);
    // 더한 정보를 MapDisPtr의 주소에 저장
    IRB.CreateStore(IncrDist, MapDistPtr)->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

    /* Increase count at shm[MAPSIZE + (4 or 8)] */
    
    // 카운터 값을 젖아할 포인터 생성
    Value *MapCntPtr = IRB.CreateBitCast(IRB.CreateGEP(MapPtr, MapCntLoc), LargestType->getPointerTo());
    // 카운터 값 로드
    LoadInst *MapCnt = IRB.CreateLoad(MapCntPtr);
    MapCnt->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

    // 카운터 값에 1을 더해서 저장함
    Value *IncrCnt = IRB.CreateAdd(MapCnt, One);
    IRB.CreateStore(IncrCnt, MapCntPtr)
        ->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

    // 거리가 0일때 (target일때 target_id에 저장)
    if (distance == 0) {
        ConstantInt *TFlagLoc = ConstantInt::get(LargestType, MAP_SIZE + 16 + target_id);
        Value* TFlagPtr = IRB.CreateGEP(MapPtr, TFlagLoc);
        ConstantInt *FlagOne = ConstantInt::get(Int8Ty, 1);
        IRB.CreateStore(FlagOne, TFlagPtr)
            ->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));
        outfile3 << target_id << " " << getDebugInfo(bb) << std::endl;
        target_id++;
    }

    // dbb에 대해서 
    if (critical_bbs.count(bb)) {
        for (auto cbb : critical_bbs[bb]) {
            BasicBlock::iterator IP2 = cbb->getFirstInsertionPt();
            llvm::IRBuilder<> IRB2(&(*IP2));

            // CBMap 에서 값을 로드함
            LoadInst *CBPtr = IRB2.CreateLoad(CBMapPtr);
            CBPtr->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

            // bb, 1에 해당하는 값을 생성
            ConstantInt *CBIdx = ConstantInt::get(Int32Ty, bb_id);
            ConstantInt *CBOne = ConstantInt::get(Int8Ty, 1);
            
            //CBPtr에서 CBIdx 떨어진 주소의 포인터를 계산함.
            Value* CBIdxPtr = IRB2.CreateGEP(CBPtr,CBIdx);

            // 그 주소에 1을 저장
            IRB2.CreateStore(CBOne, CBIdxPtr)
                ->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));
            c_instrument_num++;
        }
        critical_ids[bb] = bb_id;
    }

    if (solved_bbs.count(bb)) {
        for (auto sbb : solved_bbs[bb]) {
            BasicBlock::iterator IP2 = sbb->getFirstInsertionPt();
            llvm::IRBuilder<> IRB2(&(*IP2));

            // CBMap에서 값을 로드
            LoadInst *CBPtr = IRB2.CreateLoad(CBMapPtr);
            CBPtr->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));

            // bb, 2에 해당하는 값 생성
            ConstantInt *CBIdx = ConstantInt::get(Int32Ty, bb_id);
            ConstantInt *CBOne = ConstantInt::get(Int8Ty, 2);
            
            // 주소 계산
            Value* CBIdxPtr = IRB2.CreateGEP(CBPtr,CBIdx);

            // 그 주소에 2를 저장
            IRB2.CreateStore(CBOne, CBIdxPtr)->setMetadata(M->getMDKindID("nosanitize"), MDNode::get(*C, None));
        }
    } 
```


> solved BB와 critical BB사이의 차이가 무엇일까

> taint_bbs 개념

`cbi.cpp:countCFGDistance`
``` cpp
for (BasicBlock* bb : target_bbs) {
		FIFOWorkList<BasicBlock*> worklist;
		std::set<BasicBlock*> visited;
		worklist.push(bb);
		while (!worklist.empty())
		{
			BasicBlock* vbb = worklist.pop();
            // taint_bbs는 target으로부터의 predecessors
			tmp_taint_bbs.insert(vbb);
			for (BasicBlock* srcbb : predecessors(vbb)) {
				if (visited.find(srcbb) == visited.end() && !isCircleEdge(LoopInfo,srcbb,vbb)) {
					worklist.push(srcbb);
					visited.insert(srcbb);
				}
			}
		}
	}

	taint_bbs[svffun->getLLVMFun()] = tmp_taint_bbs;
```

> solved vs critical

`cbi.cpp:identifyCriticalBB`
``` cpp
for (SVFModule::iterator iter = svfModule->begin(), eiter = svfModule->end(); iter != eiter; ++iter){
	const SVFFunction* svffun = *iter;
	for (Function::iterator bit = svffun->getLLVMFun()->begin(), ebit = svffun->getLLVMFun()->end(); bit != ebit; ++bit) {
		BasicBlock* bb = &*(bit);
		std::set<BasicBlock*> tmp_critical_bbs;
		std::set<BasicBlock*> tmp_solved_bbs;
		if (!bb->getSingleSuccessor() && taint_bbs.count(svffun->getLLVMFun()) && taint_bbs[svffun->getLLVMFun()].count(bb)) {
			for (BasicBlock* dstbb : successors(bb)) { 
				if (taint_bbs[svffun->getLLVMFun()].count(dstbb) == 0) {
					tmp_critical_bbs.insert(dstbb); // bb의 successors(dstbb)가 taint 되지 않았더라면 아니라면
				}
				else {
					tmp_solved_bbs.insert(dstbb); // taint되어 있다면 > solved_bbs 
				}
			}
		}
		if (!tmp_critical_bbs.empty()) { // 만약 successor가 전부 taint되어 있다면 저장하지 않음
			critical_bbs[bb] = tmp_critical_bbs;
			solved_bbs[bb] = tmp_solved_bbs;
		}
	}
}
```




# 4. calculated_cb_distance
critical_bits[i] == 1 or 2 가 의미 

1 : critical_bbs
2 : solved_cbbs

왜 solved_cbbs = critical_bbs일까
