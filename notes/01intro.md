# Compiler Construction

Marcin Benke, MIM UW

Lecture 1 - Introduction

October 5, 2025

## A Riddle

```c
int collatz(int n) {
    while(n != 1) {
      if(n % 2 == 0)
        n /= 2;
      else
        n = 3 * n + 1;
    }
    return n;
}
```

Answer me these questions three:

- does `collatz` always return 1?
- what is the best (correct) code a compiler can generate?
- what is the air-speed velocity of an unladen swallow?


### Surprising solution

x86-64 clang 14 generates the following:

```
collatz:
    mov     eax, 1
    ret
```

Preserves program semantics according to C language standard.

Infinite loops without observable effects are **undefined behaviour**.

### Another Mystery

```c
int sumto(int n) {
  int i=0, sum=0;
  while (i<n) {
    i = i+1;
    sum = sum+i;
 }
 return sum;
}
```

How many times does the loop roll? How many jumps are taken?

### There is no ~~spoon~~ loop

```nasm
sumto:  test    edi, edi
        jle     .L1
        lea     eax, [rdi - 1]
        lea     ecx, [rdi - 2]
        imul    rcx, rax
        shr     rcx
        lea     eax, [rcx + 2*rdi]
        add     eax, -1
        ret
.L1:
        xor     eax, eax
        ret
```

### What is a compiler?

A program translating programs in higher-level language to a machine language, be it for a hardware processor (e.g. x86, ARM)
or a virtual machine (e.g. JVM),
**preserving program semantics**.

Difference between interpreters and compilers:

- an interpreter executes programs,
- a compiler does not, it just translates them;<br/>
  (it can execute parts of the program, e.g. Template Haskell, comptime in Zig or Solidity)
- writing an interpreter does not require familiarity with the target machine,
- creating a compiler requires intimate familiarity with the target machine.
- creating a good compiler requires familiarity
  with the quirks of the source language (UB etc.)

### What does a compiler do?

- Reads an input program, usually as text.
- Checks correctness and *analyses* the program.
- Thinks for a while (performing a number of program transformations).
- Generates target code *(synthesis)*.

The compiler parts responsible for analysis and synthesis are called
*front-end* and *back-end* respectively.

### Analysis

Examine the program text and represent it in a form convenient for later phases (usually some form of an annotated structure tree):

- lexical analysis --- breakdown into lexemes ("words");
- syntax analysis --- breakdown of the program structure and its representation as a tree *(syntax tree)*
- semantic analysis --- binding identifier uses with proper declarations;
  type checking etc.

**Every phase must provide clear error messages**.

Hard to do but really important.

The analysis phase is mostly independent of the target language.

### Example

Let us consider a simple function in a C-like language:

```c
int sumto(int n)
{
  int i, sum;
  i = 0;
  sum = 0;
  while (i<n) {
    i = i+1;
    sum = sum+i;
 }
 return sum;
}
```

### Lexical analysis

The input gets split into *lexemes*:

![](lexemes.svg)

<!--
```
[int] [id sumto] [(] [int] [id n] [)] [{]
[int] [id i] [,] [id sum] [;]
[id i] [=] [num 0] [;]
[id sum] [=] [num 0] [;]
[while] [(] [id i] [<] [id n] [)] [{]
[id i] [=] [id i] [+] [num 1] [;]
[id sum] [=] [id sum] [+] [id i] [;]
[}] [return] [id sum] [;] [}]
```
-->

### Syntax analysis

Concrete syntax (program text) contains elements needed only to ease unambiguous read of it: parentheses, semicolons, indentation etc

Compilers work on abstract syntax with no such elements.

Syntax analysis builds an *Abstract Syntax Tree* (AST) from lexem stream:

![](while-ast.svg)

### Semantic (static) analysis

- Declaration analysis
- Information stored in the *symbol table*
- Symbol use correctness check and binding to respective declarations
  (via the symbol table).
- Type checking (or inference).

### Synthesis

- Translation of the syntax tree into a form suitable for further transformations<br/>
(called IR - *Intermediate Representation* or  IL - *Intermediate Language*).
- IR transformations
- Code improvement ("optimisation")
- Instruction choice
- Instruction scheduling
- Register allocation
- Target code generation

### Target architectures

- A physical processor architecture, e.g. x86, x86\_64, ARM
- A virtual machine
  - stack-based, e.g. JVM
  - register-based, e.g. LLVM
- A virtual machine can be used as an intermediate stage towards real machine code
  - *Ahead of Time (AOT)* --- machine code is generated before the start of the execution (e.g. LLVM)
  - *Just in Time (JIT)* --- machine code is generated during execution, (e.g. JVM).

### Stack machine

Operation arguments and results on the stack

- (+) Easy code generation from a syntax tree, e.g.

  ```haskell
  genIntExp (EBinOp e1 op e2) = do
     genIntExp e1
     genIntExp e2
     intOp op

  intOp "+" = emit "iadd"
  ```

- (-) Hard to optimise
- (-) Real processors are not stack machines.

### `sumto` for a stack machine (JVM)

```
.method public sumto()I
  iconst_0
  istore_2
  iconst_0
  istore_3
 L1: iload_2
     iload_1
     if_icmpge L2
     iload_2
     iconst_1
     iadd
     istore_2
     iload_3
     iload_2
     iadd
     istore_3
     goto  L1
 L2: iload_3
     ireturn
 .end method
```

### Register machine

**Registers**

hold data inside the CPU

- (+) access an order of magnitude faster than to memory
- (-) scarce; e.g. x86 has 7 universal registers

**LLVM**

register-based, unbounded number of (virtual) registers

can be mapped onto actual CPU by means of *register allocation*

### `sumto` compiled to 80x86 assembly (GNU as)

```att
         .globl  sumto
        .type   sumto, @function
sumto:
.LFB0:
        testq   %rdi, %rdi      # if n < 0 goto .L4
        jle     .L4
        movl    $0, %edx        # rdx = 0 (sic!)
        movl    $0, %eax        # rax = 0
.L3:
        addq    $1, %rax        # rax += 1
        addq    %rax, %rdx      # rdx += rax
        cmpq    %rax, %rdi
        jne     .L3             # if rax != n goto .L3
.L1:
        movq    %rdx, %rax      # return rdx
        ret
.L4:
        movl    $0, %edx        # rdx = 0
        jmp     .L1

        # stack is not executable
        .section        .note.GNU-stack,"",@progbits
```

### `sumto` for LLVM

```llvm
entry:
  %0 = icmp sgt i32 %n, 0
  br i1 %0, label %bb.nph, label %L5
bb.nph:
  %tmp4 = add i32 %n, -2
  %tmp2 = add i32 %n, -1
  %tmp5 = zext i32 %tmp4 to i33
  %tmp3 = zext i32 %tmp2 to i33
  %tmp6 = mul i33 %tmp3, %tmp5
  %tmp7 = lshr i33 %tmp6, 1
  %tmp8 = trunc i33 %tmp7 to i32
  %tmp = shl i32 %n, 1
  %tmp9 = add i32 %tmp, %tmp8
  %tmp10 = add i32 %tmp9, -1
  ret i32 %tmp10
L5:  ret i32 0
```

### The nanopass approach

The classical approach to writing (and teaching) compilers is

- front to backend
- whole language at a time
- working compiler only at the end

The nanopass approach:

- start with the backend for a small language
- always have a working compiler
- grow the language by adding features
- every pass has a single, well-defined purpose

### Course Organisation

This edition of the course is  lab-centered.

**Lecture** introduces important concepts and techniques.

**Tutorials** are an opportunity to enhance your familiarity with these techniques an practice them on paper/blackboard examples.

**Lab project:** write a compiler for a simple language to x86_64 assembly.

Not one big project, but a progression of simple languages

- Lvar: arithmetic expressions + assignment
- Lwhile: conditionals and loops
- Lfun: toplevel functions
- extensions:
    - tuples, garbage collection
    - lexically scoped functions, lambdas
    - optimisations

### Submission and attribution policy

All code not authored by yourself (Stackoverflow, LLMs etc)
**must be clearly attributed**.

Use of LLMs is **strongly** discouraged, except for really menial tasks - you will not learn as much.

**You have to be able to explain every piece of submitted code** (not being able to do so, or failing to present the code => 0p)

Substantial parts of the code in the lecture notes and lab support materials have been adapted by permission from Jeremy Siek's book *Essentials of Compilation*.


### Grading

- Exam 40%
- Lab projects 40%
- Midterm (colloquium) 20%

Lab projects must be submitted and presented by the dates indicated on moodle.<br/>
Failure to submit or satisfactorily present the solution => 0p.

For first exam admission at least 50% points from lab+midterm required
(30% required for the reexam).

Exam alternative: students who get at least 70% points from lab+midterm (and midterm >50%),
may choose to get a grade based on their $1.6*n$.
(i.e. as if their exam marks were equal to $0.6*n$);
where n = lab+midterm points

According to department rules, participation in tutorials and laboratory sessions, is mandatory, and unexcused absence from more than 20% of the sessions may result in the forfeiture of the right to pass the course.

### Course timeline

- Week 1-2 - translating expressions
- Week 3 - register allocation
- Week 4-5 - conditionals, loops
- Week 6 - functions (milestone)
- Week 7 - tuples, GC
- Week 8 - advanced functions
- Week 9 - optimisations
- Week 10-11 - parsing
- Week 12 - midterm

### Submission calendar

All project submissions need to be presented at the lab according to the calendar below.</br>

Independently, solutions need to be submitted to moodle.

| Project      | Presentation   | Late Presentation | Moodle | Points
|--------------|----------------|-----------------| -------- | ------
| Lqua         | Week 2         | Week 3          | Oct 15   |  2
| Lvar         | Week 3         | Week 4          | Oct 20   |  2
| reg alloc    | Week 4         | Week 5          | Oct 27   |  5
| loops        | Week 6         | Week 7          | Nov 10   |  5
| functions    | Week 8         | Week 10         | Nov 24   | 10
| extensions   | Week 11        | Week 12         | Dec 15   | up to 16


Note: approximate dates, binding dates on Moodle.

**No submissions after the Moodle cut-off date ("ostateczny termin") will be accepted**

### Resources

- `https://moodle.mimuw.edu.pl/`
    - key for lab group n: `Mrjp26#n` e.g. `Mrjp26#9` for group 9
- Jeremy Siek --- *Essentials of Compilation*, MIT Press 2023, PDF freely available.
- Cooper, Torczon --- *Engineering a Compiler*, Elsevier 2012
- Lecture notes appear after each lecture on **moodle.mimuw.edu.pl**
- My email: **ben@mimuw.edu.pl**
- Office hours by appointment; (official: Mon 14:30)

**Ask questions!**

## The target machine: x86-64

### x86-64 --- the subset we need for starters

```
reg   ::= rsp | rbp | rax | rbx | rcx | rdx | rsi | rdi
        | r8 | r9 | r10 | r11 | r12 | r13 | r14 | r15
dst   ::= %reg | int(%reg)
src   ::= $int | dst
instr ::= addq src, dst | subq src, dst | negq dst
        | movq src, dst | pushq dst | popq dst
        | callq label   | retq
prog  ::= .globl main
          main: instr*
```

- 16 general-purpose 64-bit registers.
- `$n` is an *immediate*;
- `n(%r)` is memory at address `r + n`, e.g. `-16(%rbp) ~ M[%rbp - 16];
- n (without $) means M[n] so `movq 0, %rax` is usually bad;<br/> `movq %rax, 0` potentially even worse;
- Suffix `q` = quadword = 64 bits; in most cases can be omitted,>br/> but the book and GNU tools use it.


### Instruction semantics

- `movq s, d` --- copy `s` to `d`
- `addq s, d` --- `d := d + s` (**in place**, two operands only)
- `subq s, d` --- `d := d - s`
- `negq d`    --- `d := -d`
- `pushq s`   --- push `s` (usually a register) on the stack
- `popq d`    --- pop top 8 bytes off the stack into  `d`
- `callq f`   --- push return address, jump to `f`
- `retq`      --- pop return address and jump to it

We will discuss stack in a moment.

### Restrictions

Crucial restriction:

> **At most one operand of an instruction may be a memory reference.**

Rare corner-case:

> if d is in memory,and s is immediate, then it is limited to 32 bit (signed)

(`movq $4294967296, %rax` is allowed, `movq $4294967296, -8(%rbp)` is forbidden)

### x86 Instructions Example

``` att
pushq %rbp
movq %rbp, %rsp      # %rbp -> %rsp
subq $16, %rsp       # %rsp -= 16
movq %rdi, 8(%rbp)   # %rdi -> memory[%rbp + 8]
callq input_int
movq 8(rbp), %rdi
addq %rax, %rdi
callq print_int
movq %rbp, %rsp
popq %rbp
retq
```

### Beyond the book --- multiplication

The book's *Lvar* has only `+`, `-` and unary minus. Our labs add `*`.

`imulq` is *not* analogous to `addq`:

- its destination must be a **register**
- there is different form for multiplication by (32 bit) constant


```att
imulq %rcx, %rax             # ok
imulq -8(%rbp), %rax         # ok --- the source may be memory
imulq $13, -8(%rbp)          # ERROR: operand size mismatch
imulq $137, %rdx, %rax       # OK, RAX := 137 * RDX
```

### A first x86 program

Computing `10 + 32`:

```att
    .globl main
main:
    movq $10, %rax
    addq $32, %rax
    retq
```

`main` returns the value in `%rax` to the operating system as the
*exit code*.
```
$ make answer
cc    answer.s   -o answer
$ ./answer
$ echo $?
42
```

Here we get 42, but exit code 0 conventionally means success<br/>
--- so we will
normally set `%rax` to 0 at the end and call a function to print results.

### Using memory

Using variables: `x = 10; x = -x; return x + 52`

```att
    .globl main
main:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp
    movq    $10, -8(%rbp)
    negq    -8(%rbp)
    movq    -8(%rbp), %rax
    addq    $52, %rax
    addq    $16, %rsp
    popq    %rbp
    retq
```

The intermediate result lives on the stack;<br/>
%rbp is commonly used to address local/temporary variables

### The stack

The *procedure call stack* consists of a *frame* per active call.

- `%rsp` --- stack pointer, top of stack
- `%rbp` --- base pointer, used to address the current frame
- the stack **grows downwards**: we allocate by *subtracting* from `%rsp`

| Position     | Contents       |
|--------------|----------------|
| `8(%rbp)`    | return address |
| `0(%rbp)`    | old `%rbp`     |
| `-8(%rbp)`   | variable 1     |
| `-16(%rbp)`  | variable 2     |
| ...          | ...            |
| `0(%rsp)`    | variable *n*   |



### Prelude and conclusion


Prelude:

```att
pushq %rbp          # save the caller frame pointer
movq %rsp, %rbp     # set the new frame pointer
subq $N, %rsp       # reserve space for local variables
```

Conclusion:

``` att
addq $N, %rsp      # free local variables
popq %rbp          # restore caller frame pointer
retq               # return to caller
```

### main function

Wrap the instruction sequence in `main`:

```att
    .globl main
main:
    pushq %rbp
    movq  %rsp, %rbp
    subq  $N, %rsp        # N = frame size, multiple of 16
    ...                   # the compiled program
    movq  $0, %rax        # exit code 0
    addq  $N, %rsp
    popq  %rbp
    retq
```

### Calling functions
We will deal with functions later, for now we have to handle reading and printing

Here's a program that reads an integer and prints it back

``` att
  .globl main
main:
    push %rbp
    callq input_int      # result in %rax
    movq %rax, %rdi      # 1st arg in %rdi
    callq print_int
    popq %rbp
    retq
```

`input_int` and `print_int` are defined in a C runtime file; function call protocol (ABI) tells how to call them.

Building:
```
$ make fun
gcc -g -o fun fun.s runtime.c -Wl,-z,noexecstack
```
### What's wrong with this program?

``` att
  .globl main
main:
    callq input_int      # result in %rax
    movq %rax, %rdi      # 1st arg in %rdi
    callq print_int
    retq
```

seems legit, but when we try to run it:

```
$ make error
gcc -g -o error error.s runtime.c -Wl,-z,noexecstack

$ ./error
Segmentation fault
```

### Stack alignment

x86-64 requires `%rsp` to be a multiple of **16** immediately before
every `callq`.

- `callq main` pushed an 8-byte return address, so on entry `%rsp` is
  8 off;
- `pushq %rbp` pushes another 8 --- now aligned again;
- therefore the amount we subtract must itself be a multiple of 16.

One variable needs 8 bytes, but we still subtract 16.
Three variables need 24 bytes --- we subtract 32.

Getting this wrong often leads to crashes inside `printf`, which are
confusing to debug.


## Conditional execution

We will not need these for the first three labs, but for the future:

| Instruction        | Meaning                                        |
|--------------------|------------------------------------------------|
| `cmpq s2, s1`      | compare; result goes to **EFLAGS**             |
| `set<cc> d`        | byte `d` := 1 if EFLAGS matches `cc`, else 0        |
| `movzbq s, d`      | move a byte register into a 64-bit destination (zero extend) |
| `jmp label`        | unconditional jump                             |
| `j<cc> label`      | jump if EFLAGS matches `cc`                    |
| `xorq s, d`        | exclusive or --- we use it to implement `not`  |

`<cc>` - condition codes: `e` `ne` `l` `le` `g` `ge`.

New argument kind: **byte registers** `al`, `bl`, `cl`, `dl` --- the
low bytes of `rax`, `rbx`, `rcx`, `rdx`.

### set<cc> --- and the byte register

`set<cc>` writes **one byte** --- its destination must be a byte
register:

```att
cmpq  $1, %rcx
sete  %al                # al := (rcx == 1)
movzbq %al, %rdx         # rdx := zero-extend(al)
```

- `movzbq` (*move, zero-extend, byte to quad*) extends the value to 64-bit register<br/> (important, otherwise we may get garbage).
- `%al` sits inside `%rax`, so writing it clobbers part of `%rax`. Our
  read/write sets say `%rax`, not `%al`.

# Questions?
