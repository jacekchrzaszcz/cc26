# Compiler extensions

Choose some extensions. Note that some extension may depend on other ones
(e.g. to allow more than 6 arguments using tuples, you need to implement tuples),
but these dependencies are not common.

## Move biasing (3p)

- Implement move biasing as described in EoC Chapter 4.7 (1p)

- Extend move biasing to cover registers as well (2p),<br/>
so that for example

```python
def add(x: int, y: int) -> int:
    return x + y
```

produces

```att
    movq %rdi, %rax
    addq %rsi, %rax
    jmp add_conclusion
```

and not

```att
    movq %rdi, %rcx
    movq %rsi, %rdx
    movq %rcx, %rax
    addq %rdx, %rax
    jmp add_conclusion
```

Add flags to turn it on/off:

- `--fmove-biasing / --fno-move-biasing`

## Tuples, Garbage collector (up to 6p)

- implement tuples without garbage collector support (2p)

- implement compiler support for the garbage collector described in EoC Chapter 7. (4p)

- alternatively: implement a reference counting GC (1p)

For the allocator/collector yourself you can use the code provided in `support/runtime-full.c`


## Functions (up to 16p)

### more arguments (up to 3p)

allow more than 6 arguments by using tuples as described in EoC Chapter 8 (2p)

alternatively

allow more than 6 arguments by passing arguments on the stack (3p)


### first class functions (4p)

allow functions to be assigned to variables, passed as function arguments, returned as results
as described in EoC Chapter 8

```
def map(f : Callable[[int], int], v : tuple[int,int]) -> tuple[int,int]:
    return f(v[0]), f(v[1])
    
def inc(x : int) -> int:
    return x + 1

inc1 = inc
print(map(inc1, (0, 41))[1])

### lexically scoped functions (6p)

### efficient tail calls (3p)

Implement efficient tail calls as described in EoC 8.2.2

### closure optismisation (3p)

## Parsing (3p)

- implement a LALR Lark parsers for Lvar and Lfun
(including transforming Lark parse tree to respective Python AST subset);
add an option use your parser in the compiler instead of `ast.parse`
(note: both methods should be available to demonstrate that results are the same).
- design a C-style syntax (braces and semicolons instead of indents) for Lfun
- write a program translating between Python-style and C-style programs (both ways)

## Optimisations

Note: tail call elimination is described in the functions section.
### SSA (preparatory step)

Many transformations are easier on the SSA form.
Add a pass to your compiler to transform Cfun programs to SSA form.

### Folding/propagation

- constant folding, constant propagation, copy propagation, e.g.

``` python
x = 1
y = 2 + x
z = y * y
t = z
print(t)
```

should be transformed to `print(9)`, however

```
x = 1
x = input_int()
print(x)
```

should NOT be converted to `print(1)` (SSA helps)

Add flags to turn them on/off:

- `--fpropagate / --fno-propagate`
- `--fcfold / --fno-cfold`

### LCSE/GCSE (1-2p)

Add passes performing LCSE and GCSE transformations. Add flags to turn them on/off:

- `--flcse / --fno-lcse`
- `--fgcse / --fno-gcse`

### Partial/comptime evaluation

Some calls to functions that have no side-effects can often be evaluated at compile time.
For example

```python
def fib(n: int) -> int:
    if n < 2:
        return n
    else:
        return fib(n - 1) + fib(n - 2)

print(fib(10))
```

can be folded to `print(55)`

Be careful to avoid side-effects and not to throw the compiler into infinite loop/recursion.


### Block rearrangement (1-2p)

In our code, every block is terminated by a jump. Some of these jumps can be removed by rearranging blocks, e.g.

```att
main_start:
    movq $0, sum
    movq $5, i
    jmp loop.0

block.1:
    movq sum, %rdi
    callq print_int
    movq $0, %rax
    jmp main_conclusion

block.2:
    addq i, sum
    subq $1, i
    jmp loop.0

loop.0:
    cmpq $0, i
    jg block.2
    jmp block.1

main_conclusion:
    ...
```

Can be rearranged to


```att
main_start:
    movq $0, sum
    movq $5, i
    # fall through to loop.0
loop.0: 
    cmpq $0, i
    jg block.2
    # fall through too block.1
block.1:
    movq sum, %rdi
    callq print_int
    movq $0, %rax
    jmp main_conclusion

block.2:
    addq i, sum
    subq $1, i
    jmp loop.0

main_conclusion:
    ...
```

Often there are multiple possible arrangements - try to find the best one and argue for your choices.
Remove unreachable blocks (e.g. produced by `while False` etc)
For best results - measure runtimes.


## Submission

Present your work on the 11th or 12th lab. If you implement more than one extensions, you may present them
ins a couple of sessions - as they are completed.

Submit your work on moodle (as tar.gz) before presenting.

Late penalty: 20% (up to 3p) per week started.

Moodle cut-off date: Jan 6 (after this date, no submissions accepted)
