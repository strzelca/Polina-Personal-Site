+++
title = "EmuCPU: a CPU that doesn't exist, in C"
date = "2025-09-14"
+++

The EmuCPU is a emulator for, something that doesn't exist, it's not a real architecture,
I wrote this for fun, because I had nothing to do.
It's simple, two registers, pointers, flags. Arithmetic operations are made through bitwise operations.

<!--more-->

# Architecture

The architecture it's nothing special, the idea is a RISC ISA, a simple fetch-decode-execute cycle with no pipelines. The implementation in C is a struct defining the prototype.

```C
typedef struct _cpu {
      uint16_t a;
      uint16_t b;
      uint16_t pc;
      uint16_t sp, bp;
      union flags {
              // flags implementation
      } flags;
} cpu_t;
```

We have four two-byte registries. The A and B general purpose registries, so they can be used for everything during execution. After that we have the PC (program counter) and both SP (stack pointer) and BP (base pointer), this enables set up and alignment of the stack.

A special attention should be paid to the flags:

```C
union flags {
    uint8_t Z; // zero
    uint8_t S; // sign
    uint8_t O; // overflow
    uint8_t C; // carry
    uint32_t flags;
} flags;
```

Flags are an union of four one-byte length flag: Zero, Sign, Overflow and Carry.
Since I didn't want to use the flags one by one, I put them into an union and I use them as the four-byte flags entry.

# Instruction Set

The Instruction Set actually provides 9 instructions. I tried to implement them not as a +/-/* but with a real binary logic. A summary is:

* **ADD(*dst*, *v1*, *v2*, *flags*)**: dst <- v1 + v2
* **INC(*dst*, *flags*)**: dst++
* **SUB(*dst*, *v1*, *v2*, *flags*)**: ADD(dst, v1, -v2, flags)
* **MUL(*dst*, *v1*, *v2*, *flags*)**: while(v2--) ADD(dst, dst, v1, flags)
* **DIV(*dst*, *v1*, *v2*, *flags*)**: while (v1 not 0) SUB(v1, v1, v2, flags),INC(dst, flags)
* **MOV(*dst*, *n*v)**: dst <- n
* **PUSH(*src*, *bp*, *mem*)**: mem->mem[\*bp] <- *src, *bp++
* **POP(*dst*, *bp*, *mem*)**: *dst <- mem->mem[\*bp] <- , mem->mem[\*bp] <- 0
* **ALIGN(*sp*, *mem*)**: *sp <- mem->size
