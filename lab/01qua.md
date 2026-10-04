# 01 The language *Lqua*

The language *Lqua* (called $L^{mon}_{var}$ in Jeremy Siek's book)
is described by the following abstract syntax (we use a subset of Python abstract syntax defined in the standard module `ast`):

```
prog ::= Module(list[stmt])
stmt ::= Expr( Call(Name('print'), [atm]) ) | Assign([var], exp)
exp ::= atm
      | Call(Name='input_int', [])
      | UnaryOp(USuB(),atm)
      | BinOp(atm,Add(),atm)
      | BinOp(atm,Sub(),atm)
atm ::= Constant(int) | var
var ::= Name(str)
```

A program is a sequence of statements, each of which may be an assignment or a `print` statement

An expression can be an atom, a call to `input_int`, or a unary/binary operation on atoms.

An atom is an integer literal or a variable name

Note:
- Python treats `print` as an expression
- `Expr` is a Python class name; `exp` is a nonterminal
- `[exp]` is a list containing exactly one expression


## Tasks

### Before the lecture

1. Familiarise yourself with Python `ast` module; check how abstract syntax trees for various constructs are built, e.g.

```
>>> dump(parse('x+1'))
"Module(body=[Expr(value=BinOp(left=Name(id='x', ctx=Load()), op=Add(), right=Constant(value=1)))])"
```

2. Write an interpreter for *Lqua* (and perhaps *Lvar*)

3. Refresh your knowledge of x86 assembly; try writing some simple programs.

### After the lecture

4. Manually translate some simple *Lqua* programs to x86 assembly, putting variables on the stack; do not try to prematurely optimise anything, instead think how an automated translation might look like. Use 64-bit integers/registers.

 For example

 `a = input_int(); mone = -1; b = a - mone; print(b)`

might yield

```
   .globl main
main:
    pushq %rbp
    movq %rsp, %rbp
    subq $32, %rsp
    callq input_int
    movq %rax, -24(%rbp)
    movq $1, %rax
    negq %rax
    movq %rax, -16(%rbp)
    movq -24(%rbp), %rax
    subq -16(%rbp), %rax
    movq %rax, -8(%rbp)
    movq -8(%rbp), %rdi
    callq print_int
    movq %rbp, %rsp
    movq $0, %rax
    popq %rbp
    retq
```
Note that stack must be aligned to 16 bytes, hence `subq $32, %rsp` even if the variables only use 24.

### The runtime

```c
long input_int() {
    long x;
    int count = scanf("%ld", &x);
    if (count) return x;
    else { puts("input_int failed!\n\n"); return 0; }
}

void print(long x) {
    printf("%ld\n", x);
}
```

Compile and link with the generated assembly:

```
gcc -g -o prog prog.s runtime.c -Wl,-z,noexecstack
```

without the final option pack we may get a warning:

```
ld: warning: missing .note.GNU-stack section implies executable stack
```

### Homework

write a translator from *Lqua* to x86; follow the schema in `support/compiler_skel.py`:
- `remove_complex_expressions` does nothing yet, it will be used in the future;
- `select_instructions` chooses x86 instructions for *Lqua* operations; for example
```
b = a - 1               =>     movq a,  %rax
                               subq $1, %rax
                               movq %rax, b
```
it is OK if the generated instructions contain variables, because
- `assign_homes` replaces these variables by addresses of stack locations, e.g.
```
movq a,  %rax           =>     movq -8(%rbp)
subq $1, %rax                  subq $1, %rax
movq %rax, b                   movq %rax, -16(%rbp)
```
Do not worry about inefficiency of storing all variables on the stack; we will deal with this soon enough.
- `patch_instructions` replaces instructions where both operands refer to memory,which is not allowed on x86; e.g. `subq -16(%rbp), -24(%rbp)` becomes
```
movq -24(%rbp), %rax
subq -16(%rbp), %rax
movq %rax, -24(%rbp)
```
- `prelude_and_conclusion` adds prelude and conclusion, e.g.

```
    pushq %rbp
    movq %rsp, %rbp
    subq $32, %rsp
    ...
    movq %rbp, %rsp
    movq $0, %rax
    popq %rbp
```
The `movq $0 %rax` is optional but adding it in the `main` function may make testing easier, since return value from `main` becomes the program exit code and anything other than 0 can be interpreted as failure.

You can use x86 abstract syntax provided in `support/x86_ast.py` (for this task `Instr`, `Callq` and `X86Program` should be enough).


## Final checks

Write some tests for your translator; make sure all key functionalities are covered.

- generated code assembles and runs correctly
- subtraction, negation
- number of stack slots used, stack alignment
- `input_int`

## Submission

Present your work on the next lab (2p) or a week later (1p).

Submit a single `<uid>.tar.gz` file to moodle, where uid is your user id on students, in the format `xy128410`

After unpacking the archive, the compiler should be runnable with `uv run compiler.py <input file>`.

## Recommended practices

Use git (or Jujutsu over git) for version management **from the start**.

Use uv for Python project management; you can point it at support files instead of copying them:

```
[tool.uv.sources]
mrj-support = { path = "../support", editable = true }
```

Use types everywhere possible; run mypy often and heed its warnings.

Use `match` to handle abstract syntax trees (visitor pattern is sometimes useful too).

``` python
def select_arg(self, a:expr) -> arg:
    match a:
        case Constant(value):
            match value:
                case int():
                    return ...
                case _:
                    raise ...
        case Name(id):
            return ...
        case _:
            raise ...
```


