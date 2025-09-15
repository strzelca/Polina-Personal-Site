+++
title = "EmuCPU: a CPU that doesn't exist, in C"
date = "2025-09-14"
+++

The EmuCPU is a emulator for, something that doesn't exist, it's not a real architecture,
I wrote this for fun, because I had nothing to do.
It's simple, two registers, pointers, flags. Arithmetic operations are made through bitwise operations.

<!--more-->

# Architecture

The architecture it's nothing special, the idea is a **RISC ISA**, a simple _fetch-decode-execute_ cycle with no pipelines. The implementation in C is a struct defining the prototype.

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

We have **four _two-byte_ registries**. The A and B general purpose registries, so they can be used for everything during execution. After that we have the PC (program counter) and both SP (stack pointer) and BP (base pointer), this enables set up and alignment of the stack.

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

Flags are an union of **four _one-byte_ length flag**: _Zero, Sign, Overflow and Carry_.
Since I didn't want to use the flags one by one, I put them into an union and I use them as the four-byte (_u32_) flags entry.

# Instruction Set

The **Instruction Set** actually provides 9 instructions. I tried to implement them not as a _+/-/*_ but with a real binary logic. There is a concept, without the bitwise operations, only the idea, I was _"functionally"_ inspired ad I wrote it in [COQ](https://rocq-prover.org).

```coq
(* Addition: ADD *)
Definition ADD (dst v1 v2: u16) (flags: u32) : instruction :=
    fun op =>
        let res := v1 + v2 in
        set_reg dst result op.
        

(* Increment: INC *)
Definition INC (dst : u16) (flags : u32) : instruction :=
  fun op =>
    let val := get_reg dst op in
    let res := val + 1 in
    set_reg dst res op.

(* Subtract: SUB *)
Definition SUB (dst v1 v2 : u16) (flags : u32) : instruction :=
  ADD dst v1 (neg v2) flags.

(* Multiply by repeated addition: MUL *)
Definition MUL (dst v1 v2 : u16) (flags : u32) : instruction :=
  fun op =>
    while (get_reg v2 op > 0) do
      let op1 := ADD dst dst v1 flags op in
      let op2 := set_reg v2 ((get_reg v2 op) - 1) op1 in
      op2.

(* Divide by repeated subtraction: DIV *)
Definition DIV (dst v1 v2 : u16) (flags : u32) : instruction :=
  fun op =>
    while (get_reg v1 op > 0) do
      let op1 := SUB v1 v1 v2 flags op in
      let op2 := INC dst flags op1 in
      op2.

(* Move immediate: MOV *)
Definition MOV (dst n : u16) : instruction :=
  fun op =>
    set_reg dst n op.

(* PUSH to stack *)
Definition PUSH (src bp : u16) (mem : memory) : instruction :=
  fun op =>
    let addr := get_reg bp op in
    let val  := get_reg src op in
    let op1  := set_mem addr val op in
    let op2  := set_reg bp (addr + 1) op1 in
    op2.

(* POP from stack *)
Definition POP (dst bp : u16) (mem : memory) : instruction :=
  fun op =>
    let addr := get_reg bp op in
    let val  := get_mem addr op in
    let op1  := set_reg dst val op in
    let op2  := set_mem addr 0 op1 in
    op2.

(* ALIGN stack pointer *)
Definition ALIGN (sp : u16) (mem : memory) : instruction :=
  fun op =>
    let size := mem_size mem in
    set_reg sp size op.

```
