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
shm_id2
# 4. sniff_mask


# 5. fuzz/llvm_mode/afl-llvm-pass.so.cc

# 6. instrument/src/cbi.cpp
