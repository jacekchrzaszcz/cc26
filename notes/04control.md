# Booleans, Conditionals and Loops

Marcin Benke, MIM UW

Compiler Construction --- lecture 4

## Roadmap

### Where we left off

So far we have: integer arithmetic, straight-line code.

![](control-pipeline.svg)

One basic block per program --- the language had no way to *not*
execute an instruction.

### What we are adding

Control flow, in two parts:

- **`Lif`** --- booleans, comparisons, `not`/`and`/`or`,
  `if` statements and conditional expressions.
- **`Lwhile`** --- `while` loops.

```python
x = input_int()
if x > 0 and x < 10:
    print(1)
else:
    print(0)
```

```python
sum = 0
i = 5
while i > 0:
    sum = sum + i
    i = i - 1
print(sum)
```

Most of the work in the first one, loops change liveness analysis and not much else.

### Two new problems

1. **A second kind of value.** Booleans share the 64-bit registers
   with integers.<br/>
   What are `True` and `False`? What does `x < y`
   compile to?

2. **Control flow.** A list of instructions becomes a *graph* of
   blocks.<br/>
   Every pass must be revisited --- and liveness, a
   single backward sweep, breaks outright.

Loops make the latter harder: the graph has cycles.

### The pipeline

![](lif-pipeline.svg)

- `shrink` --- new;
- `explicate_control` --- new, and the heart of the lecture;
- everything else --- an extension of a pass we have to multiple blocks.

### Lif --- what is new

Concrete syntax, on top of `Lvar`:

```
cmp ::= == != < <= > >=
exp  ::= ... | True | False
       | exp and exp | exp or exp | not exp
       | exp cmp exp                 
       | exp if exp else exp
stmt ::= ... | if exp: stmt+ else: stmt+
```

Abstract syntax: free, from Python's `ast`:

```
Constant(True) | Constant(False)
BoolOp(And()|Or(), [exp, ...])       -- note: a list, not a pair
UnaryOp(Not(), exp)
Compare(exp, [cmpop], [exp])         -- note: lists again
IfExp(test, body, orelse)
If(test, [stmt], [stmt])
```

## The target: x86If

### New instructions

| Instruction        | Meaning                                        |
|--------------------|------------------------------------------------|
| `cmpq s2, s1`      | compare; result goes to **EFLAGS**             |
| `set<cc> d`        | `d` := 1 if EFLAGS matches `cc`, else 0        |
| `movzbq s, d`      | move a byte register into a 64-bit destination (zero extend) |
| `jmp label`        | unconditional jump                             |
| `j<cc> label`      | jump if EFLAGS matches `cc`                    |
| `xorq s, d`        | exclusive or --- we use it to implement `not`  |

`<cc>` - condition codes: `e` `ne` `l` `le` `g` `ge`.

New argument kind: **byte registers** `al`, `bl`, `cl`, `dl` --- the
low bytes of `rax`, `rbx`, `rcx`, `rdx`.

### cmpq --- three quirks

```att
cmpq %rbx, %rax        # sets flags as if computing  rax - rbx
```

1. **Operand order** `x < y` is `cmpq y, x` (as in `subq y, x`)
2. **Result invisible.** In EFLAGS, which no instruction can name;
   only `set<cc>` and `j<cc>` read it.
3. **Second operand may not be an immediate.**<br/>
   `cmpq $1, $2` does not
   assemble --- `patch_instructions`' job (or eliminate it in
   `shrink`).

<!--
X86 AST: `Instr('cmpq', [arg2, arg1])` for `Compare(arg1, [cmpop], [arg2])`.
-->
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

### Basic blocks

x86 has no nested statements --- only labels:

```att
main_start:
    callq input_int
    movq  %rax, x
    cmpq  $1, x
    je    block.1
    jmp   block.2
block.1:
    movq  $42, _1
    jmp   block.0
```

A **basic block**: one entry (the label), one exit (the jump at the
end). Control never enters or leaves in the middle.

So `X86Program.body` becomes a `dict[str, list[instr]]` --- must handle this shape from now on.

### The two ways to use a comparison

A comparison can be needed either as a **value**:

![](if-cmp1.svg)

<!--
```python
x = a > b
```

```att
cmpq   b, a
setg   %al
movzbq %al, x
```
-->
or as a **condition**

![](if-cmp2.svg)

<!--
```python
if a > b: ... else: ...
```

```att
cmpq b, a
jg   block_then
jmp  block_else
```
-->

Conflating them costs three instructions:

```att
cmpq   b, a
setg   %al
movzbq %al, _1
cmpq   _1, 1       # Compare again ??
je     block_then
jmp    block_else
```

Getting the compiler to *choose* the right one is the role of `explicate_control`.

### shrink: and / or are not operators

`and/or` are short-circuit --- conditionals, not operations on
booleans:

```python
e1 and e2   ==>   e2   if e1 else False
e1 or  e2   ==>   True if e1 else e2
```

`shrink` rewrites them up front; downstream only ever sees `IfExp`.

## remove_complex_operands

### Where an if may appear

`IfExp` is complex, so in atomic position it gets a temporary:

```python
print(1 + (10 if x > 0 else 20))
```
⟹
```python
_1 = 10 if x > 0 else 20
_2 = 1 + _1
print(_2)
```

Unremarkable. The interesting cases are where `Lif^mon` is
*deliberately* more liberal.

### The condition stays complex

The test of an `if` is **not** made atomic:

```python
if (x == 0 if x < 1 else x == 2):
    ...
```

- a temporary means materialising a 0/1 in a register --- the code we
  are trying to avoid;
- left complex, `explicate_control` sees its shape and picks jumps.

The operand of `not`, almost the same:

- a **comparison** is kept whole --- `not (a > b)` stays in one piece;
- anything else (`not (a and b)`, already an `IfExp`) gets a
  temporary.<br/>
- possibly better alternative for not: handle in shrink, not in the book




### Branch temporaries are local

```python
print((input_int() + 1) if x > 0 else 0)
```

The body needs a temporary for `input_int()`. \

Hoisted out of the
branch, we would read a number even when the condition is false.

So each branch temporaries are wrapped up *with* it.<br/>
Only the test temporaries escape --- the test always runs.

### Begin

RCO returns an atom plus the **statements** binding its temporaries.<br/>
Normally they go in front of the current statement;<br/>
a branch of an `IfExp` has nowhere to put them --- it is an *expression*

So we add one new form:

- `Begin(stmts, result)` runs the statements,
then produces the value.
- `rco_exp` builds it:

![](if-input.svg)

<!--
```
    x = input_int()
    _2 = ({
        _1 = input_int()
        produce (_1 + 1)
    } if x > 0 else 0)
    print(_2)
```
-->

where `{ ... produce ...}` is invented concrete syntax for `Begin`<br/>
(only for diagnostic IL printouts)

### Begin does not last long
`Begin` is not Python syntax; it lives only in our IL.

The `explicate_control` pass *removes* it, moving
statements out of expressions and into blocks:

```
block.1:
    _1 = input_int()
    _2 = (_1 + 1)
    goto block.0
block.2:
    _2 = 0
    goto block.0
```

A functional language would use `let ... in ...` (or maybe `do { ... }`)<br/>

Python has no expression that binds a variable, so we add one.

## explicate_control

### The problem

After RCO: a *tree* of `if`s. The target: a *graph* of blocks.

```python
x = input_int()
y = input_int()
tmp = y + 2 if (x == 0 if x < 1 else x == 2) else y + 10
print(tmp)
```

An `if` whose condition is itself an `if` --- where does the branch for
`x < 1` jump to?

### The naive translation

Each comparison and each `if` in isolation:

```att
cmpq   $1, x
setl   %al
movzbq %al, tmp
cmpq   $1, tmp
je     then_branch
jmp    else_branch
```

Materialise, then compare *that* against 1: correct, and three
instructions of pure ceremony.<br/> What we want:

```att
cmpq $1, x
jl   then_branch
jmp  else_branch
```

The comparison must sit **immediately before** the jump that uses it.

### Reorganising trees is not the solution

One way to achieve this goal is to reorganize the code at the level of LIf , pushing
the outer if inside the inner one, yielding the following code:
```
x = input_int()
y = input_int()
print(((y + 2) if x == 0 else (y + 10)) \
    if (x < 1) \
    else ((y + 2) if (x == 2) else (y + 10)))
```

Unfortunately, this approach duplicates the two branches from the outer if.

This is no good - even if it looks reasonable at the first sight,<br/>
it may lead to exponential blowup of the code size!


### CIf --- the target language

```
atm  ::= int | var | bool
exp  ::= atm | input_int() | -atm | atm+atm | atm-atm
       | not atm | atm cmp atm
stmt ::= var = exp | print(atm) | atm
tail ::= return exp | goto L | if atm cmp atm: goto L else: goto L
CIf  ::= { L: stmt* tail, ... }
```

A block is statements ending in a **tail**; the only control flow is
`goto L`.<br/>
No nesting anywhere --- one step from assembly.

In our code: `CProgram(body: dict[str, list[stmt]])`; `Goto` and
`Begin` from `cif.py`, the rest from Python's `ast`.

### Graphs  of blocks, not instruction trees

The example becomes:

```
main_start:                            block.4:
    x = input_int()                        if x == 2:
    y = input_int()                          goto block.2
    if x < 1:                              else:
      goto block.3                           goto block.1
    else:                              block.2:
      goto block.4                         tmp = y + 2
block.3:                                   goto block.0
    if x == 0:                         block.1:
      goto block.2                         tmp = y + 10
    else:                                  goto block.0
      goto block.1                     block.0:
                                           print(tmp)
                                           return 0
```

Six blocks for three `if`s. `block.2`, `block.1` and `block.0` each
have **two predecessors**<br/>
--- a graph, not a tree, and nothing duplicated.

### Four functions, one per position

| function | handles |
|---|---|
| `explicate_stmt` | a statement, followed by `cont` |
| `explicate_assign` | an expression for its **value** --- into `lhs` |
| `explicate_effect` | an expression for its **effect** --- value discarded |
| `explicate_pred` | an expression as a **condition** --- one of two tails |

Mutually recursive, all four growing `basic_blocks` as a side effect.

They differ on `IfExp`: the same expression compiles differently in
each position. 

### Continuations, shared not copied

- `cont` --- "the code that runs after this", already compiled.
- Back to front is what makes it work: at an `if` the continuation
  exists already, so both branches get the *same* one.
- The base case is `return 0` --- (`main` returns 0, we will have other returns soon)

The rule that keeps the output linear in the input:

> A continuation used more than once must be a block, and each use a
> `goto`.

Inline it into both branches instead, and `n` conditionals give $2^n$
copies of the tail.

### Example 1 --- an if expression

```python
x = input_int()
print(42 if x == 1 else 0)
```

after `shrink` and RCO:

```python
x = input_int()
_1 = 42 if x == 1 else 0
print(_1)
```

after `explicate_control`:

```
main_start:                 block.1:
    x = input_int()             _1 = 42
    if x == 1:                  goto block.0
      goto block.1          block.2:
    else:                       _1 = 0
      goto block.2              goto block.0
                            block.0:
                                print(_1)
                                return 0
```

`block.0` is the shared continuation; `_1` is assigned in each branch.

(we try to mimic the source order, block *creation* order is
`block.0` first, `main_start` last.)

### Example 2 --- nested if statements

```python
a = 1
b = 2
if a > 0:
    if b > 0:
        print(1)
    else:
        print(2)
else:
    print(3)
```

```
main_start:                          block.1:
    a = 1                                print(1)
    b = 2                                goto block.0
    if a > 0:                        block.2:
      goto block.3                       print(2)
    else:                                goto block.0
      goto block.4                   block.4:
block.3:                                 print(3)
    if b > 0:                            goto block.0
      goto block.1                   block.0:
    else:                                return 0
      goto block.2
```

Three `print`s, three blocks, one shared `return`; `block.0` exists before anything that
jumps to it.

### Example 3 --- and

```python
x = input_int()
if x > 0 and x < 10:
    print(1)
else:
    print(0)
```

`shrink` turns the test into

```python
x < 10 if x > 0 else False
```

and then:

```python
main_start:                          block.2:
    x = input_int()                      print(1)
    if x > 0:                            goto block.0
      goto block.3                   block.1:
    else:                                print(0)
      goto block.1                       goto block.0
block.3:                             block.0:
    if x < 10:                           return 0
      goto block.2
    else:
      goto block.1
```

Two comparisons, each immediately before its jump; `block.1` shared by
both failure paths;<br/>
`False` never materialised. Short-circuit
evaluation ensured by `shrink` plus `explicate_pred`.

## select_instructions

### The new statements

Per block now, but otherwise business as usual:

```
goto L                      =>  jmp L
```

```
return e                    =>  movq e, %rax
                                jmp main_conclusion
```

```
if a cmp b: goto L1         =>  cmpq b, a
else: goto L2                   j<cc> L1
                                jmp L2
```

`If` can occur only in the form `If(Compare(...), [Goto], [Goto])` ---
`explicate_control` guarantees it.

### Reminder --- Example 1: shrink, RCO, explicate

```python
x = input_int()
print(42 if x == 1 else 0)
```

after `shrink` and RCO:

```python
x = input_int()
_1 = 42 if x == 1 else 0
print(_1)
```

after `explicate_control`:

```
main_start:                 block.1:
    x = input_int()             _1 = 42
    if x == 1:                  goto block.0
      goto block.1          block.2:
    else:                       _1 = 0
      goto block.2              goto block.0
                            block.0:
                                print(_1)
                                return 0
```

### Example 1, all the way down
```python
x = input_int()
print(42 if x == 1 else 0)
```


```att
main_start:                                          main_start:
    x = input_int()                                      callq input_int
    if x == 1:                                           movq %rax, x
      goto block.1                                       cmpq $1, x
    else:                                                je block.1
      goto block.2                                       jmp block.2

block.1:                                             block.1:
    _1 = 42                                              movq $42, _1
    goto block.0                                         jmp block.0

block.2:                                             block.2:
    _1 = 0                                               movq $0, _1
    goto block.0                                         jmp block.0

block.0:                                             block.0:
    print(_1)                                            movq _1, %rdi
    return 0                                             callq print_int
    movq $0, %rax
    jmp main_conclusion
```

Two variables left to place; the shape is already x86.

## Register allocation with branches

### The single-block sweep is not enough

Last lecture: one block, one backward pass, `L_after(n) = {}`.

Now every block has a successor, and

```att
    jmp block.n
```

is an instruction whose live-after depends on `block.n` live-before
set.

Two questions:

1. In which order do we process the blocks?
2. What are the read and write sets of `jmp` and `j<cc>`?

### The control-flow graph

Nodes: labels. Edges: jumps --- one per `Jump`/`JumpIf`, straight out
of the block dictionary.

![](branch-cfg.svg)

<!--
```
   main_start
     /     \
 block.1  block.2
     \     /
     block.0
        |
  main_conclusion
```
-->

The way we generate code, a `j<cc>` is always followed by a `jmp`<br/>
--- so a conditional block has exactly two successors.

### In which order?

![](branch-cfg.svg)

- `main_start` liveset depends on `block.1` and `block.2`;
- these two in turn depend on `block.0`

This suggests the following order:

- `block.0`
- `block.1` and `block.2` in any order
- `main_start`

In general, a block depends on the live-before sets of its successors, so:<br/>
successors first --- the **reverse topological order** of the CFG, i.e. the
topological order of its transpose.

Topological order needs an acyclic graph, annd without loops the graph is acyclic.

We will handle loops in a moment.

### Analysing the jumps

A jump reads nothing and writes nothing. Its *live-after* set comes
from a successor **block**:

```
L_after(jmp L)   = live_before_block[L]
L_after(j<cc> L) = live_before_block[L] U L_before(next)
```

- `j<cc>` may go either way: assume both, take the **union**.
  Conservative costs a register; wrong costs correctness.

<!--
- `L_before(k) = (L_after(k) - W(k)) U R(k)` is unchanged; with `R`
  and `W` empty it copies `L_after` back.
-->


For example for the sequence

```att
cmpq $1, x
je block.1
jmp block.2
```

we have

```
L_after(cmpq $1, x) = live_before_block[block.1] U live_before_block[block.2]
```

### R and W for the new instructions

| Instruction     | R                     | W               |
|-----------------|-----------------------|-----------------|
| `cmpq s2, s1`   | `s1`, `s2`            | *nothing*       |
| `set<cc> %al`   | *nothing* (the flags) | `rax`           |
| `movzbq %al, d` | `rax`                 | `d`             |
| `xorq s, d`     | `s`, `d`              | `d`             |
| `jmp L`, `j<cc> L` | *nothing*          | *nothing*       |

- A jump's contribution is that live-after set, not `R`.
- The flags are not a location we allocate, so they never appear ---
  never reorder a `cmpq`/`set<cc>` pair.
- `%al` is a part of `%rax`

### Prelude and conclusion with blocks

Both become blocks of their own:

```att
main:
    pushq %rbp
    movq  %rsp, %rbp
    subq  $16, %rsp
    jmp   main_start        # <-- we could also make prelude fall into start and save the jump
...
main_conclusion:
    movq  %rbp, %rsp
    popq  %rbp
    retq
```

`main` is the entry block; every finishing path jumps to the
conclusion.

`Lif` has no `return` of its own --- the one `Return` is the base
continuation, so there is exactly one `jmp main_conclusion`.<br/>

If callee-saved registers are used, we must save them in prelude and restore in the conclusion;<br/>
with multiple rturns this code would be duplicated.

## Loops

### Lwhile

One new statement:

```
stmt ::= ... | while exp: stmt+
```

```
While(test, [stmt], [])       # Python's ast; the [] is the else clause
```

- type checker: the test must be `bool`.

New with `while`: an unbounded number of executions of the same
instruction. <br/>
*Placing* values now matters far more than it did.

### A test's temporaries belong to the test

The test runs again on every iteration. So where do *its* temporaries
go?

```python
while i - 1 > 0:
    ...
```

In front of the loop, `i - 1` would be computed once --- for the first
test only.<br/>
They have to stay **inside** the test, and `Begin` is again
what holds them.

An `if` is different: its test temporaries can be hoisted in front of
the statement.

Hence `explicate_pred` now must handle `Begin` as well.

### while needs no new intermediate language

`CIf` already has `goto` and conditional jumps --- all a loop is:

```
loop:
    if test: goto body else: goto after
body:
    ...
    goto loop
after:
    ...
```

Add one case to `explicate_control`; its output language does not change
at all.


One novelty: the loop's label
must exist **before** the body is compiled,<br/> because the body jumps
back to it (and vice versa). Generate labels first, fill content later.

### The cycle

Our loop from the start of the lecture, after instruction selection
(labels shortened for readability):

![](loop-cfg.svg)

`after` is fine: live-before `{sum}`.<br/>
But `loop` needs `body` and
`body` needs `loop` --- no order puts successors first, because there
is no topological order.

Original source:
```python
sum = 0
i = 5
while i > 0:
    sum = sum + i
    i = i - 1
print(sum)
```

### The cycle - livesets

![](loop-cfg.svg)

We get a recursive set of equations:

```
live_after(after)      = {}
live_after(loop)       = live_before(after) U live_before(body)
live_after(body)       = live_before(loop)
live_after(main_start) = live_before(loop)

live_before(after)     = live_after(after) U {sum}
live_before(loop)      = live_after(loop) U {i}
live_before(body)      = live_after(body) U {i, sum}
```

Can be solved by iterating to a fixpoint.

### Computing the fixpoint

Start with the **empty** live-before set for every block --- an
*underapproximation*. Then repeat:

1. recompute each block's live-after from the current guesses for its
   successors;
2. sweep the block backwards for a new live-before;
3. stop when a round adds nothing.

Each round only *adds*, and variables are finite, so it stops.

### The iteration

Live-before sets, one column per block.

$m_0$ --- start with empty sets:

```
main_start: {}      loop: {}        body: {}       after: {}
```

$m_1$ --- perform one sweep per block, using $m_0$ for the successors:

```
main_start: {}      loop: {i}       body: {i,sum}  after: {sum}
```

$m_2$ --- `loop`'s live-after is now the union of what `body` and
`after` want, `{i,sum} ∪ {sum} = {i,sum}`:

```
main_start: {}      loop: {i,sum}   body: {i,sum}  after: {sum}
```

$m_3$ --- no changes in live-before sets. Done.

What the last round adds: `sum` is live in `loop`, a block that
never mentions `sum`. One sweep would miss that.

### Converged

At the fixed point, for our loop:

- `i` and `sum` are live across the whole loop, so they interfere and
  need two registers --- which is right;
- neither is live across a call, so both can be caller-saved.

(The book's sets also contain `rsp`; ours do not.)

### Now put a call in the loop

```python
n = input_int()
acc = 0
while n > 0:
    acc = acc + input_int()
    n = n - 1
print(acc)
```

Both are carried round the loop *and* live across `callq input_int`,
so neither may have a caller-saved register<br/> --- `acc` gets `%rbx`, `n`
gets `%r12`, and the prelude saves them:

```att
main:                      block.2:
    pushq %rbp                 callq input_int
    movq  %rsp, %rbp           movq %rax, %rcx
    pushq %rbx                 addq %rcx, %rbx
    pushq %r12                 subq $1, %r12
```

### Summary

- Two ways to compile a comparison: for its **value**, or for a
  **branch**. Preferring the second is what `explicate_control` is
  *for*.
- `shrink` removes forms the target lacks (`and`, `or`), not forms it
  has (`-`).
- `explicate_control` turns a tree into a graph, back to front,
  passing continuations;
- Branches make liveness a **dataflow** problem: reverse topological
  order while acyclic, **fixpoint iteration** once there are loops.


## Questions?

### Lab tasks

1. `shrink`: `and`/`or` ⟹ `IfExp`. Mind the multi-operand `BoolOp`.
2. `remove_complex_operands` for `IfExp`, `Compare`, `not`. Add
   `Begin`; keep each branch's temporaries inside it. Extend the ANF
   checker first.
3. `explicate_control`: the four functions plus `create_block`. Test
   each separately --- especially `explicate_pred` on all seven shapes
   of condition.
4. `select_instructions`: `Goto`, `Return`, `If`, comparisons both
   ways, `not` both ways.
5. `patch_instructions` and `prelude_and_conclusion` (jump to
   `_start`, conclusion as a block).
6. End-to-end tests green with `assign_homes`, before liveness --- a
   working `Lif` compiler.
7. Build the CFG, then `uncover_live` over it in reverse topological
   order, maintaining `live_before_block`. Handle `jmp` and `j<cc>`.
8. `while`: interpreter, RCO (the test's temporaries stay in the
   test), `explicate_control`.
9. Replace the sweep with iteration to fixpoint; check your loop
   programs get registers, not stack slots.
10. Print your CFGs --- and keep a test whose fixpoint takes three
    rounds.

## Extra material

### Further reading

- Kildall (1973) --- dataflow analysis and the worklist algorithm.
  Kleene (1952) for the fixed-point theorem it rests on.
- Allen (1970) --- control-flow graphs and basic blocks.
- Dybvig's course notes (2010) --- `expose-basic-blocks`, the direct
  ancestor of `explicate_control`.
- Danvy (2003), Appel (2003) --- the same translation as a conversion
  to continuation-passing style. `cont` is not called a continuation by
  accident.
- Peyton Jones & Santos (1998) --- the *case-of-case* transformation:
  GHC does to `case` what `explicate_pred` does to `if`.
- Aho et al., *Compilers* ch. 9 --- dataflow analysis at length, if you
  want the lattice theory properly.


### Why iteration to fixpoint works --- lattices

Dress it up, and the reason becomes a theorem.

- Abstract state: a set of locations, ordered by ⊆; bottom = ∅, join =
  ∪. A lattice.
- Whole-program state: a mapping *label → set*, ordered pointwise.
  Also a lattice; bottom sends every label to ∅.

One round of the analysis is a function $f$ on the second:

$$f(m_i) = m_{i+1}$$

A solution is a mapping $f$ reproduces: $f(m_s) = m_s$, a **fixed
point**. We want the *least* one --- only what the program forces to
be live.

### Why it terminates

**Kleene's fixed-point theorem.** For monotone $f$ --- better input,
better output --- the least fixed point is the limit of

$$\bot \sqsubseteq f(\bot) \sqsubseteq f(f(\bot)) \sqsubseteq \cdots$$

and with no infinite ascending chains the chain reaches it:
$f^k(\bot) = f^{k+1}(\bot)$ for some $k$.

Liveness qualifies: monotone, finitely many variables and blocks. It
*must* converge, to the right answer.

This is **dataflow analysis** (Kildall, 1973); liveness is one
instance.

### The generic worklist algorithm

Do not redo every block each round --- only those whose input changed:

```python
def analyze_dataflow(G, transfer, bottom, join):
    trans_G  = transpose(G)
    mapping  = dict((v, bottom) for v in G.vertices())
    worklist = deque(G.vertices())
    while worklist:
        node = worklist.pop()
        inputs = [mapping[v] for v in trans_G.adjacent(node)]
        input = reduce(join, inputs, bottom)
        output = transfer(node, input)
        if output != mapping[node]:
            mapping[node] = output
            worklist.extend(G.adjacent(node))
    return mapping
```

Four parameters: the graph, a per-block analysis, `bottom` and `join`.
Nothing in it mentions liveness --- which is the point, and also why
`graph.py` does not ship it: writing it is task 9.

### Instantiating it for liveness

| Parameter  | For liveness                                        |
|------------|-----------------------------------------------------|
| `G`        | the **transpose** of the CFG                        |
| `transfer` | `(label, live_after) -> live_before` for that block  |
| `bottom`   | `set()`                                             |
| `join`     | set union                                           |

- `analyze_dataflow` is written *forward*: inputs come from
  predecessors in `G`. Liveness is *backward*, so we hand it the
  transposed CFG, where "predecessors" are the real successors.
- `transfer` is your `block_live`, plus recording each instruction's
  live-after set for `build_interference_graph`. Only the final round's
  sets matter --- and that round changed nothing.

### One live_after per block?

`analyze_dataflow` joins all of a block's successors into one input.
The rule two slides back is finer: each jump takes its own successor's
set.

The difference lands on the trailing `jmp`, whose live-after becomes
the union instead of just its target's set. A `jmp` writes nothing, so
it adds no interference edge --- and the block's live-before comes out
the same either way.

Which is why `block_live` can simply read `live_before_block[L]` at
each jump, and needs no joined input at all.

### Forward and backward, may and must

The same machinery, with different parameters, is most of classical
optimisation:

| Analysis             | Direction | Join | Answers                      |
|----------------------|-----------|------|------------------------------|
| liveness             | backward  | ∪    | may this value still be read |
| reaching definitions | forward   | ∪    | which assignment reached here |
| available expressions| forward   | ∩    | is `a+b` already computed     |
| very busy expressions| backward  | ∩    | will `a+b` surely be computed |

Union gives *may* analyses (true on some path), intersection *must*
(true on all paths). Swap the lattice and you get constant propagation,
interval analysis, type inference for dynamic languages…

The one algorithm you are about to write is the skeleton of all of
them.

### Cost

One sweep per block per round: $O(\text{instructions})$. The number of
rounds is bounded by the longest ascending chain --- at worst the
number of variables.

So $O(V \cdot I)$; in practice a handful of rounds, the worklist
confining the work to what changed. Loop nesting depth is the honest
predictor, not program size.



## Implementation hints

### is_anf

The output language is small enough to check, so we check it:

```python
assert is_anf(simplified), "The AST is not in A-normal form ..."
```

`ANFChecker` insists the operands of `BinOp`, `UnaryOp`, `Compare` and
`Call` are `Name`s or `Constant`s --- bar the one exception RCO relies
on, a `Compare` under `not`. Twenty lines, and a mysterious crash in
`select_instructions` becomes an assertion at the pass that caused it.

Write the checker before the pass.

### Representing booleans

```
True  =>  1
False =>  0
```

What x86 comparisons produce, and what Python's `bool` already is (a
subclass of `int`). One consequence in `select_arg`:

```python
case Constant(value):
    match value:
        case bool():   # bool subclasses int, so this must come first
            return Immediate(1 if value else 0)
        case int():
            return Immediate(value)
```

Get the order wrong and `True` becomes `$True`. The assembler will tell
you, eventually.


### Temporaries that must not move

A branch of an `IfExp` keeps its own, wrapped by `sequence`:

```python
new_ifexp = IfExp(test_atom,
                  sequence(body_temps, body_atom),
                  sequence(orelse_temps, orelse_atom))
```

A `while` test keeps its own, in *predicate* position:

```python
case While(test, body, []):
    new_test, test_temps = self.rco_exp(test, False)
    ...
    if test_temps:
        new_test = Begin([...bind test_temps...], new_test)
```

### create_block

Two branches sharing a continuation must not mean *copying* it --- give
it a label and jump to it:

```python
def create_block(stmts, basic_blocks):
    match stmts:
        case [Goto(l)]:
            return stmts                       # already a jump; reuse it
        case _:
            label = generate_name('block')
            basic_blocks[label] = stmts
            return [Goto(label)]
```

Two points worth noting:

- it returns a **`Goto`**, so the caller can use the result freely;
- the `[Goto(l)]` case stops us wrapping a jump in a block containing
  only that jump. Without it, block counts explode.

### explicate_something

```python
# all four add to basic_blocks and return list[stmt]

explicate_stmt(s: stmt, cont: list[stmt], basic_blocks)

explicate_assign(rhs: expr, lhs: expr, cont: list[stmt],
                 basic_blocks)
    """lhs := rhs; rhs may be an ANF expr, an IfExp or a Begin."""

explicate_effect(e: expr, cont: list[stmt], basic_blocks)
    """e in ANF, compiled for its side effects, sequenced with cont."""

explicate_pred(cnd: expr, thn: list[stmt], els: list[stmt],
               basic_blocks)
    """cnd in ANF; thn and els already terminated."""
```

### explicate_stmt and explicate_assign

```python
case If(test, body, orelse):
    goto_cont = create_block(cont, basic_blocks)
    new_body   = explicate_stmts(body,   goto_cont, basic_blocks)
    new_orelse = explicate_stmts(orelse, goto_cont, basic_blocks)
    return explicate_pred(test, new_body, new_orelse, basic_blocks)
```

```python
case IfExp(test, body, orelse):
    goto_cont = create_block(cont, basic_blocks)
    then_branch = explicate_assign(body,   lhs, goto_cont, basic_blocks)
    else_branch = explicate_assign(orelse, lhs, goto_cont, basic_blocks)
    return explicate_pred(test, then_branch, else_branch, basic_blocks)
```

The same shape twice: name the continuation, compile both branches
against it, hand the test to `explicate_pred`. The assignment is pushed
**into** the branches --- `lhs` is assigned in each, and nothing
afterwards.

(`explicate_stmts` is the plural: fold `explicate_stmt` over a list,
back to front.)

### explicate_pred --- the main event

```python
def explicate_pred(cnd, thn, els, basic_blocks):
```

- `cnd` is in A-normal form;
- `thn` and `els` are already-compiled, already-terminated tails;
- the result is a tail.

Seven cases. Six ask *what is the cheapest way from this condition to
`thn` or `els`?*; the seventh is the one that keeps the other six
honest.

### explicate_pred --- comparison

The good case, and the reason for the whole design:

```python
case Compare(left, [op], [right]):
    goto_thn = create_block(thn, basic_blocks)
    goto_els = create_block(els, basic_blocks)
    return [If(cnd, goto_thn, goto_els)]
```

The comparison stays in the `If`, next to the jumps, and
`select_instructions` emits `cmpq` / `j<cc>` / `jmp`. No `set<cc>`, no
`movzbq`, no temporary.

### explicate_pred --- constants and not

```python
case Constant(True):  return thn
case Constant(False): return els
```

A two-line partial evaluator, falling out of the structure rather than
bolted on: `if True: ... else: ...` emits no test and no jump.

Also where blocks go to waste --- the discarded branch may already have
created blocks that nothing now jumps to. See the challenge at the end.

```python
case UnaryOp(Not(), operand):
    return explicate_pred(operand, els, thn, basic_blocks)
```

`not` is not compiled at all --- it swaps the two continuations. Hence
RCO keeping a **comparison** under `not` complex: a temporary would
cost `set<cc>`, `movzbq`, and a second `cmpq` to test it. Any other
operand is atomic by now and lands in the default case.

### explicate_pred --- nested if

The case that motivated the pass:

```python
case IfExp(test, body, orelse):
    elsblock = create_block(els, basic_blocks)
    thnblock = create_block(thn, basic_blocks)
    body_branch   = explicate_pred(body,   thnblock, elsblock, basic_blocks)
    orelse_branch = explicate_pred(orelse, thnblock, elsblock, basic_blocks)
    return explicate_pred(test, body_branch, orelse_branch, basic_blocks)
```

Read it as: *both branches of the inner `if` are themselves conditions
of the outer one* --- so recurse in predicate position with the outer
`thn`/`els`. Label them first (`create_block`), so both branches jump
to the same two blocks.

Same shape as the assign case, different recursive call. Position is
everything.

### explicate_pred --- Begin, and the default

```python
case Begin(body, result):
    new_result = explicate_pred(result, thn, els, basic_blocks)
    return explicate_stmts(body, new_result, basic_blocks)
```

RCO can put a `Begin` in a condition: compile the result in predicate
position, then prepend the statements.

```python
case Name() | Constant():
    return [If(Compare(cnd, [Eq()], [Constant(False)]),
               create_block(els, basic_blocks),
               create_block(thn, basic_blocks))]
```

An **atom** --- typically a variable of type `bool`. Nothing to look
at, so compare against `False`; note the branches are **swapped**,
because we test for `== False`.

### explicate_pred --- and everything else

That leaves conditions that are neither a shape we recognise nor an
atom. `input_int()` is the only one this language has:

```python
if input_int():
    ...
```

It may perfectly well be a condition. What it may **not** be is an
operand of `cmpq` --- a call is two instructions. So bind it, and ask
again:

```python
case _:
    temp = Name(generate_name('tmp'))
    return [Assign([temp], cnd)] + explicate_pred(temp, thn, els,
                                                  basic_blocks)
```

The recursion terminates immediately: `temp` is a `Name`, so the
second call takes the case above.

### Why that case is worth getting right

It is tempting to write the atom case as the fallback and handle
`input_int()` specially. Do that and the fallback silently assumes
its condition is an atom --- true today, and wrong the moment the
language grows.

The two are not symmetric:

- **what counts as an atom is a closed set** --- `Name`, `Constant`,
  and that is all it will ever be;
- **what counts as a non-atom grows** with every language: calls next
  lecture, tuple element reads after that.

Enumerate the closed one. Then `if f(x):` and `if t[0]:` need no
change to this function at all --- which is exactly what happens.

### explicate_stmt --- while

```python
case While(test, body, []):
    goto_cont = create_block(cont, basic_blocks)
    label = generate_name('loop')
    body_ss = explicate_stmts(body, [Goto(label)], basic_blocks)
    basic_blocks[label] = explicate_pred(test, body_ss, goto_cont,
                                         basic_blocks)
    return [Goto(label)]
```

`generate_name` first, `basic_blocks[label] = ...` last: the body is
compiled against a label that does not have a block yet.

### Comparisons, twice

As a value:

```python
def select_compare(self, var, cmp, left, right):
    al = x86.ByteReg('al')
    return [ Instr('cmpq', [self.select_arg(right), self.select_arg(left)])
           , Instr('set' + self.select_jmp(cmp), [al])
           , Instr('movzbq', [al, self.select_arg(var)])
           ]
```

As a test: `cmpq` + `JumpIf(cc, l1)` + `Jump(l2)`.

`select_jmp` maps the `ast` comparison operator to a condition code
(`Eq()` ⟹ `e`, `Lt()` ⟹ `l`, …) and serves both.

### Reverse topological order?

`graph.py` already has what you need:

```python
from graph import DirectedAdjList, transpose, topological_sort
```

Keep `live_before_block: label -> LiveSet` alongside it. In reverse
topological order, every successor's entry is already there when you
need it.

### Selecting not

Two cases, because RCO left the operand complex:

```python
case Assign([var], UnaryOp(Not(), Compare(left, [cmp], [right]))):
    return self.select_compare(var, negate_cmp(cmp), left, right)
```

A negated comparison folds into the condition code: `not (a > b)` is
`setle`, not `setg` + `xorq`. `negate_cmp` swaps `Eq`↔`NotEq`,
`Lt`↔`GtE`, `Gt`↔`LtE`.

```python
case Assign([var], UnaryOp(Not(), operand)):
    ...
    flip = Instr('xorq', [Immediate(1), target])
    if source == target:
        return [flip]
    return [movq(source, target), flip]
```

`x = not x` is one instruction --- as with `simple_op` last time,
checking whether the destination already holds an operand pays.

### Interference: movzbq is a move

Used more widely, `movzbq` would extend last lecture's `mov` rule:

> for `movq s, d` (and `movzbq s, d`), add an edge `d --- v` for every
> `v` live after the instruction with `v != d` and `v != s`.

`d` and `s` hold the same value afterwards, so they may share a
register.

But we only use `set<cc> %al`, and %rax is not allocated anyway.

Everything else --- greedy colouring, saturation, spilling, the
caller-/callee-saved ordering --- is unchanged. Register allocation
does not care about branches.


### patch_instructions, extended

Two new restrictions:

```python
case Instr('cmpq', [arg1, Immediate() as arg2]):
    # cmpq cannot take an immediate as its second operand
    return [movq(arg2, temp_reg), Instr('cmpq', [arg1, temp_reg])]

case Instr('movzbq', [arg1, arg2]) if is_memory_arg(arg2):
    # movzbq can only widen into a register
    return [Instr('movzbq', [arg1, temp_reg]), movq(temp_reg, arg2)]
```

Same division of labour as before: `select_instructions` chose the
instruction, `allocate_registers` the homes, `patch_instructions`
repairs what the encoding forbids. Keeping them apart is why each stays
short.


### Stretch goal --- optimize blocks and remove jumps

Two easy wins on the block graph:

- **Dead blocks.** `explicate_pred` on `Constant(True)` throws away a
  continuation and any blocks built for it. Remove what is unreachable
  from `main_start`. (`while True:` is the shortest program that leaves
  one behind.)
- **Remove jumps.** A block ending in `jmp L` where `L` has exactly
  one predecessor: merge them.

```att
main_start:                    main_start:
    callq input_int                callq input_int
    movq %rax, %rcx        =>      movq %rax, %rcx
    jmp block.4                    movq %rcx, %rax
block.4:                           addq $2, %rax
    movq %rcx, %rax                jmp main_conclusion
    addq $2, %rax
    jmp main_conclusion
```

The move pair now adjacent is something a peephole pass can see and the
split blocks hid.

Predecessor counts come from the CFG you built for liveness.

### Types, suddenly

`Lint` and `Lvar` had one type --- no program could be ill typed.
`Lif` has two:

```python
if 1 + True:      # what is this?
    ...
x = not 42
```

So the book introduces a type checker for `Lif`: a `bool` test,
arithmetic on `int`s, agreeing `IfExp` branches.

### Do we need a type checker?

Not for the programs we care about, and we do not have one. Two
consequences:

- The input must be a program **CPython would run and agree with**.
  Our end-to-end tests use CPython as the oracle (`tests/NAME.py` is
  both input and expected output), so ill-typed programs are outside
  the tested set by construction.
- Only integers print: `print_int` is all the runtime offers.
  `print(True)` prints `1`, CPython prints `True` --- so the tests
  print integers only.

Add one if you want: self-contained, and it catches real mistakes in
your own test programs.
