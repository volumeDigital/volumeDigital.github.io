---
layout: post
title: "SUBLEQ-VM: Executing general purpose code on SIMD and SIMT machines."
---

# intro

Contemporarary GPGPUs tend to offer a SIMT execution model: Single Instruction, Multiple Thread.

This means that while the GPU provides the hardware resources to run multiple threads in parallel, those threads must all run the same instruction at the same moment.

Think about this as multiple threads, all operating within the same address space, with their own registers and CPU context, but being forced to share a program counter.

For sequences of intructions with few conditional blocks, this is perfectly adequate. For example, image processing kernels and element-wise numeric operations.

The way conditional blocks are handled is that both branches are handled, and some subset of threads know to ignore the instructions for the current block.

However, general purpose code is often extremely branchy. Furthermore, we may wish to partion the work into a pipeline, and have multiple threads managing the progression of work through each stage of the pipeline.

Wouldn't it be sweet if we could find a way to convert our SIMT machine into a traditional SMT machine?

# virtual machine

We want to run any code on any thread, but this needs to look like the _same_ sequence of instructions to the underlying hardware.

One way to achieve this is to have all threads execute a virtual machine kernel, and compile our general purpose code to the bytecode of this virtual machine.

# bytecode

There's a catch. Every thread of the SIMT machine must branch together, which ultimately means every thread must step through every possible instruction of the bytecode interpreter loop. If the bytecode instruction set has _many_ instructions, this implies that each thread will spend the vast majority of its cycles skipping code for instructions it doesn't care about.

Not ideal.

# oisc

Clearly, the the smaller the bytecode instruction set, the fewer wasted cycles per cycle of the virtual machine.

There exist instruction sets with a single instruction, and these are known as One-Instruction Set Computers [1].

A popular choice of instruction for this type of computer is the _subleq_ instruction.

```
subleq a, b, c:
    mem[b] = mem[b] - mem[a]
    if ( mem[b] <= 0 )
        goto c
```

So, if we can compile our general purpose application to `subleq`, then we can run it on any thread of the SIMT machine.

# io

Like most applications, `subleq` programs cannot interact with I/O without help.

We can provide for this by allowing `subleq` programs to make system calls. One possible mechanism is for the application to write to a pre-arranged memory region, set a flag to indicate that the region contains a message, then busy-wait until the flag is lowered.

A host runtime can then periodically check for application messages, handle them by writing the result into the pre-arranged region, then unset the flag.

# llvm

It might be possible to write an LLVM backend [3] to issue subleq instructions.


- [1] https://en.wikipedia.org/wiki/Single_instruction,_multiple_threads
- [2] https://en.wikipedia.org/wiki/One-instruction_set_computer
- [3] https://llvm.org/docs/WritingAnLLVMBackend.html