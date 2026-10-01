## Lab 4 - conditionals (part 1)

Goal: extend the compiler to handle *Lif* (Lvar + conditionals) all the way
to x86.

This is the first of two labs on control flow. By the end of this one your
compiler translates `if`/`and`/`or`/`not`/comparisons to correct x86 — one
instruction sequence per basic block, every variable on the stack. What it
does *not* do yet: register allocation for a program with more than one
block. Lecture 3's liveness-and-colouring pipeline was built for a single
straight-line block; an `if` produces several, and making liveness cross
block boundaries — plus `while`, whose control-flow graph has a cycle —
is next lab's material. For now, whenever the program has more than one
block, fall back to putting every variable on the stack (the scheme from
lecture 2).

## Lif --- what is new

Concrete syntax, on top of `Lvar`:

```
exp  ::= ... | True | False
       | exp and exp | exp or exp | not exp
       | exp cmp exp                 cmp ::= == != < <= > >=
       | exp if exp else exp
stmt ::= ... | if exp: stmt+ else: stmt+
```

We keep borrowing Python `ast`, so the abstract syntax is free:

```
Constant(True) | Constant(False)
BoolOp(And()|Or(), [exp, ...])       -- note: a list, not a pair
UnaryOp(Not(), exp)
Compare(exp, [cmpop], [exp])         -- note: lists again
IfExp(test, body, orelse)
If(test, [stmt], [stmt])
```

Add the new constructs to the interpreter.


### CIf --- the target language

```
atm  ::= int | var | bool
exp  ::= atm | input_int() | -atm | atm+atm | atm-atm
       | not atm | atm cmp atm
stmt ::= var = exp | print(atm) | atm
tail ::= return exp | goto L | if atm cmp atm: goto L else: goto L
CIf  ::= { L: stmt* tail, ... }
```

Each blocks is terminated by a return or jump (cf. LLVM IR)

You can find some abstract syntax for *Cif* in `support/cif.py`

### shrink

Write a `shrink` method implementing the transformations

```
e1 and e2   ==>   e2   if e1 else False
e1 or  e2   ==>   True if e1 else e2
```

Test it using the intepreter.

### Stretch goal

- eliminate constant comparisons such as `1 > 0` - they are cumbersome in assembly;
- rewrite `if True ...` and `if False`


### Folding a chain

Python parses `a and b and c` as a *single* `BoolOp` with three
values, so `visit_BoolOp` has to fold. Fold right, so the leftmost
operand is tested first:

```python
values = [self.visit(v) for v in node.values]
result = values[-1]
for v in reversed(values[:-1]):
    result = combine(v, result)
return result
```

`a and b and c`  ⟹  `(b and c) if a else False`  ⟹
`(c if b else False) if a else False`.

Note the recursive `self.visit(v)`: the operands may contain `and`
themselves.

## remove_complex_operands

`IfExp` is a complex expression, so where a subexpression must be
atomic it gets a temporary:

```python
print(1 + (10 if x > 0 else 20))
```
⟹
```python
_1 = 10 if x > 0 else 20
_2 = 1 + _1
print(_2)
```

The test of an `if` is **not** made atomic:

```python
if (x == 0 if x < 1 else x == 2):
    ...
```
is not translated to

```
b = x == 0 if x < 1 else x == 2
if b: ...
```

Extend `remove_complex_operands` to handle conditionals

## Explicate control

Implement functions

```python
explicate_stmt(s, cont, basic_blocks)          # a statement
explicate_assign(rhs, lhs, cont, basic_blocks) # right-hand side of =
explicate_effect(e, cont, basic_blocks)        # value discarded
explicate_pred(cnd, thn, els, basic_blocks)    # a condition
```

(alternatively: make them methods of the compiler class modifying its `basic_blocks` attribute)

### explicate_stmts

`explicate_stmts` is the fold over a statement list, and it goes
**backwards**:

```python
for s in reversed(ss):
    cont = explicate_stmt(s, cont, basic_blocks)
return cont
```

(in Haskell it would be just a `foldr` without reversing).

`explicate_control` is where the target grammar's `tail` matters: build a
dict `{label: block}`, wrap the whole program's statements in one final
`return 0`, and thread `basic_blocks` through every call so an `if`'s two
branches can share whatever comes after them instead of each getting a copy.
Test the four `explicate_*` functions separately, and check `explicate_pred`
on all of `Compare`, `Constant(True)`, `Constant(False)`, `not`, and nested
`if`.

Two cases are left over, and it pays to be careful about which is which.
A test that is an **atom** — a variable, or a constant that is not `True`
or `False` — becomes a comparison against `False`, with the branches
swapped; that is where a bool-valued variable ends up, and it also gives
`if 5:` Python's truthiness. **Anything else** is bound to a temporary
first, and the temporary tested instead: it may well be a bool, but it is
not an atom, so it cannot be an operand of `cmpq`. `if input_int():` is
the case you have already that needs this. Enumerate the atoms — there
are two — rather than the non-atoms, whose number grows with every later
lab.

A continuation used more than once must become a block of its own,
reached by `goto` — get this wrong and `n` nested `if`s produce 2ⁿ copies of
whatever comes after them.

## select_instructions

`CIf` adds three shapes on top of what `select_instructions` already knows
from *Lqua*: a comparison as the right-hand side of an assignment, `not`,
and a block-ending conditional jump.

### Comparisons

`var = left cmp right` computes the flag and materialises it as 0 or 1:

```
a = x < y
```
⟹
```
    cmpq y, x
    setl %al
    movzbq %al, a
```

Two things to get right:
- `cmpq b, a` sets the flags for comparing *a* to *b* (i.e. it computes `a - b`), so the operand order is reversed from how you'd say it: the *right*-hand side of `<` goes first.
- `set<cc>` only ever writes one byte, so its destination has to be `%al` (not `%rax`); `movzbq` then zero-extends it into the real destination.

The six comparisons, six condition codes:
```
==  e      !=  ne
<   l      <=  le
>   g      >=  ge
```

### not

Two cases, and the order they're tried in matters:

1. `var = not (left cmp right)` — do not emit the comparison and then flip
   the result; fold the negation straight into the condition code:
   ```
   a = not (x < y)     ⟹     cmpq y, x
                              setge %al
                              movzbq %al, a
   ```
   This is a special case of "comparison as right-hand side" above and must
   be tried *before* the general `not` case, or it is never reached.

2. `var = not atm`, for any other `atm` — flip the low bit:
   ```
   a = not b     ⟹     movq b, a
                        xorq $1, a
   ```
   (skip the `movq` when `a` and `b` turn out to share a home — you won't
   know that until register allocation, so leave the code able to notice)

### The conditional tail

`explicate_control` only ever produces one shape of conditional tail —
`if left cmp right: goto Lt else: goto Lf` — because `explicate_pred`
already rewrote everything else (constants, `not`, nested `if`) into
something that reduces to it. Compile that one shape and let anything else
raise:

```
if x < y: goto Lt else: goto Lf
```
⟹
```
    cmpq y, x
    jl Lt
    jmp Lf
```
using the same six condition codes as the comparisons above. A bare
`goto L` (no test) is just `jmp L`.

### Blocks

A `CProgram` is a *dict* of blocks, not a single statement list, so
`select_instructions` now runs `select_instr_body` once per block and
returns an `X86Program` whose `body` is a dict too:

```python
X86Program(body={name: self.select_instr_body(block)
                 for name, block in p.body.items()})
```

Everything downstream — `patch_instructions`, `prelude_and_conclusion` —
needs the same `isinstance(prog.body, dict)` branch alongside the old
single-block `list` case; keep both working, since a program with no `if`
still produces a single flat list of instructions the way it always did.

## Register allocation: not yet, across blocks

Lecture 3's liveness-and-colouring pipeline computes one pass over one
block's straight-line instructions. It still works unmodified on a
single-block program — one that doesn't use `if` — but nothing here asks
it to reach across a jump.

For this lab, branch on the shape of `X86Program.body` right after
`select_instructions`:

```python
if isinstance(x86_with_vars.body, list):
    # exactly the lecture-3 pipeline: uncover_live, build the
    # interference graph, allocate_registers
    ...
else:
    # more than one block - liveness across blocks isn't built yet
    assigned = self.assign_homes(x86_with_vars)
```

`assign_homes` is the lecture-2 scheme: no liveness, no colouring, every
variable gets its own stack slot. One difference from lecture 2: it needs
**one home map for the whole program**, not one per block — a variable
assigned in one block may be read in another, so blocks can't each start
numbering their slots from zero.

Making liveness reach across a control-flow graph — and then across a
*cyclic* one, once `while` exists — is the subject of the next lab.

## prelude_and_conclusion, for several blocks

With one block, prelude and conclusion were just instructions glued onto
the front and back of the list. With several blocks they become blocks of
their own:

- the block named `main` holds only the prelude (save `%rbp`, push the
  callee-saved registers the colouring used, if any, allocate stack space)
  and ends with `jmp main_start` — `main_start` is the block
  `explicate_control` built from the program's actual statements;
- a new block `main_conclusion` holds the epilogue (pop what was pushed,
  `retq`); a `return` compiles to `movq value, %rax` followed by
  `jmp main_conclusion`, from wherever it occurs.

The 16-byte stack alignment rule from lecture 2 is unchanged; it just now
has to account for the pushed callee-saved registers too (there won't be
any yet, since `assign_homes` never allocates one — but keep the code
correct for when there is).

## Checkpoint

Compile and run a program with a plain `if`/`else` (no `and`/`or`, no
nested `if`) end to end — CPython and your compiled binary should agree.
Then work through `and`/`or`, nesting, and `not`. If a comparison's
condition code comes out backwards, check the `cmpq`
operand order.

This lab is part of a bigger task and is not submitted by itself
(although you may discuss the solution with your tutor).
Submit it together with the next one (loops).
