# Riscv-V asm thread
This version is using Ripes simulation so need timer interrupt available, thread must yield cpu to scheduler by itself. Hence save context seem useless for now. I will switch to QEMU with real preemptive later.

# How it work

## Declare memory
Create memory for the thread control block (tcb). Each tcb have 2 word-size memories to store stack pointer and program counter.

```asm
tcb1: .word 0,0 # sp,pc 
tcb2: .word 0,0
tcb3: .word 0,0
tcb4: .word 0,0
```
Then, declare the stack memory 256 words (1024 byte)
```asm
stack1: .zero 256 # 1024 byte
stack2: .zero 256
stack3: .zero 256
stack4: .zero 256
```

Two words for scheduler:
- current_thread: id (0,1,2,3) for thread 1,2,3,4, respectively. 
- current_tcb   : pointer of current thread tcb

## Init memory
Init thread, save thread stack pointer and function to word 0,1 of the tcb 
```asm
# init thread 1
lui t0, %hi(stack1)
addi t0, t0, %lo(stack1)
addi t0, t0, 1024 

lui t1, %hi(thread1_fun)
addi t1, t1, %lo(thread1_fun)

lui s1, %hi(tcb1)
addi s1, s1, %lo(tcb1)

sw t0, 0(s1)
sw t1, 4(s1)
```
Repeat for thread 2,3,4:
```asm
# init thread 2
lui t0, %hi(stack2)
addi t0, t0, %lo(stack2)
addi t0, t0, 1024
lui t1, %hi(thread2_fun)
addi t1, t1, %lo(thread2_fun)
lui s1, %hi(tcb2)
addi s1, s1, %lo(tcb2)
sw t0, 0(s1)
sw t1, 4(s1)

# init thread 3
lui t0, %hi(stack3)
addi t0, t0, %lo(stack3)
addi t0, t0, 1024
lui t1, %hi(thread3_fun)
addi t1, t1, %lo(thread3_fun)
lui s1, %hi(tcb3)
addi s1, s1, %lo(tcb3)
sw t0, 0(s1)
sw t1, 4(s1)

# init thread 4
lui t0, %hi(stack4)
addi t0, t0, %lo(stack4)
addi t0, t0, 1024
lui t1, %hi(thread4_fun)
addi t1, t1, %lo(thread4_fun)
lui s1, %hi(tcb4)
addi s1, s1, %lo(tcb4)
sw t0, 0(s1)
sw t1, 4(s1)
```

Start sp and ra with thread_1's stack and function
```asm
# start with thread 1
lui sp, %hi(stack1)
addi sp, sp, %lo(stack1)
addi sp, sp, 1024 
lui ra, %hi(thread1_fun)
addi ra, ra, %lo(thread1_fun)

# start scheduler
j scheduler_loop
```

## Thread function
For basic example purpose, thread 1,2,3,4 will just increase register a3.a4,a5,a6 by 1 every loop respectively. 
**Noice:** This version is using Ripes so thread must yield cpu itself
```asm
thread1_fun:
    addi a3, a3,1
    j scheduler_loop

thread2_fun:
    addi a4, a4,1
    j scheduler_loop
    
thread3_fun:
    addi a5, a5,1
    j scheduler_loop
    
thread4_fun:
    addi a6, a6,1
    j scheduler_loop
```

## Scheduler loop
First it load current_thread id to t1 then use it to check for which thread it should switch to. If current_thread id is 0 (thread_1) switch to thread_2 and so on.

```asm
# scheduler loop
# a0 hold current_thread mem who will store thread id (0,1,2,3)

# get current thread addr first
lui a0, %hi(current_thread)
addi a0, a0, %lo(current_thread)
lw t1, 0(a0)

# choose thread
addi t0, zero, 0
beq t1,t0, switch_to_thread_2

addi t0, zero, 1
beq t1, t0, switch_to_thread_3

addi t0, zero, 2
beq t1, t0, switch_to_thread_4

addi t0,zero, 3
beq t1, t0, switch_to_thread_1
```

## Switch thread
Entry of thread, it load current_thread id which is holding in a0 first so next scheduler know thread is running. Then, load tcb of the next thread to a1.

```asm
switch_to_thread_1:
    # save current_thread id to 0 (thread1)
    addi t0, zero, 0
    sw t0, 0(a0)
    
    # load tcb1 addr to a1
    lui a1, %hi(tcb1)
    addi a1, a1, %lo(tcb1)
    
    j context_switch
    
switch_to_thread_2:
    addi t0, zero, 1
    sw t0, 0(a0)
    lui a1, %hi(tcb2)
    addi a1, a1, %lo(tcb2)
    j context_switch
    
switch_to_thread_3:
    addi t0, zero, 2
    sw t0, 0(a0)
    lui a1, %hi(tcb3)
    addi a1, a1, %lo(tcb3)
    j context_switch

switch_to_thread_4: 
    addi t0, zero, 3
    sw t0, 0(a0)
    lui a1, %hi(tcb4)
    addi a1, a1, %lo(tcb4)
    j context_switch
```

## Context switch
This is where thread function be called, ai is pointing to next thread tcb. It save the pointer sp, ra to tcb of old thread first so that thread will know where it stopped. Then load tcb of next thread to sp, ra and run. Also update current tcb for next scheduler loop.

```asm
context_switch:
    # load addr of current_tcb
    lui a0, %hi(current_tcb)
    addi a0,a0 %lo(current_tcb)
    # get the addr of current tcb to a0
    lw a0, 0(a0)
    
    # save current context
    sw sp, 0(a0)
    sw ra, 4(a0)
    
    # load next context
    lw sp, 0(a1)
    lw ra, 4(a1)
    
    # update current tcb pointer
    lui t0, %hi(current_tcb)
    addi t0, t0, %lo(current_tcb)
    sw a1, 0(t0)
    
    ret # jump to next thread stack
```