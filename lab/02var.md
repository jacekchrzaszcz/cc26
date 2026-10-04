# 02 The language *Lvar*

1. Add multiplication and integer division to *Lqua*:

```
exp ::= ...
      | BinOp(atm,Mul(),atm)
```

**Stretch goal:**

add division and modulo:

```
exp ::= ...
      | BinOp(atm,Mul(),var)
      | BinOp(atm,FloorDiv(),var)
      | BinOp(atm,Mod(),atm)
```

Note: division on x86 is a bit weird:
- dividend in rdx:rax
- divisor in register or memory (not immediate arg)
- quotient in rax, remainder in rdx

For example the following code computes `7 // 2` and `7 % 2`:

```
	movq $7, %rax
	movq $2, %rcx
	cqo
	idiv %rcx
	pushq %rdx
	movq %rax, %rdi
	callq print_int
	popq %rdi
	call print_int
```

2. The language *Lvar* extends *Lqua* by allowing complex expressions:

```
prog ::= Module(list[stmt])
stmt ::= Expr( Call(Name('print'),[exp] ) | Assign(var,exp)
exp ::= atm
      | Call(Name('input_int'))
      | UnaryOp(USuB(),exp)
      | BinOp(exp,Add(),exp)
      | BinOp(exp,Sub(),exp)
      | BinOp(exp,Mul(),exp)
atm ::= Constant(int) | var
var ::= Name(str)
```

Add a `remove_complex_expressions` pass to your compiler, which will effectively translate *Lvar* to *Lqua*.

In the process you will will need to introduce temporary variables for intermediate results.

For example

```
a = input_int()
b = input_int()
c = input_int() + 1
print(b*b - 4*a*c)
```

becomes

```
a = input_int()
b = input_int()
_1 = input_int
c = _1 + 1
_2 = b * b
_3 = 4 * a
_4 = _3 * c
_5 = _3 - _4
print(_5)
```
(using `_i` for variable names  is a suggestion based on the assumption such names are not used in the source program; if you want to completely eliminate the collison risk, you may use names that are not valid variable names such as `%tmp1` or a separate AST node type)

## Final checks
Write some tests for your translator; make sure all key functionalities are covered.

- multiplication
- generating temporary names
- complex expressions

## Submission

Present your work on the 3rd (2p) or 4th lab (1p).

Submit your work on moodle (as tar.gz) before presenting.

Submit a single `<uid>.tar.gz` file to moodle, where uid is your user id on students, in the format `xy128410`.

After unpacking the archive, the compiler should be runnable with `uv run compiler.py <input file>`.

## Recommended practices

Use git (or Jujutsu over git) for version management **from the start**.

Use uv for Python project management you can point it at support files instead of copying them:

```
[tool.uv.sources]
mrj-support = { path = "../support", editable = true }
```

## Stretch goal

Ensure the resulting code is in *Static Single Assignment* (SSA) form: every variable is assigned exactly once.

For example

```
a = input_int()
a = a + 1
print(a)
```

becomes

```
_a1 = input_int()
_a2 = _a1 + 1
print(_a2)
```
