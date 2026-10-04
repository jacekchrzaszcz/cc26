# Flow analysis and register allocation

1. Add an `uncover_live` pass (placed after `select_instructions`) computing sets of variables after each instruction, e.g.

```
  callq input_int: La = {%rax}
  movq %rax, a: La = {a}
  callq input_int: La = {a, %rax}
  movq %rax, b: La = {a, b}
  callq input_int: La = {a, b, %rax}
  movq %rax, _1: La = {a, _1, b}
  movq _1, c: La = {a, c, b}
  addq $1, c: La = {a, c, b}
  movq b, _2: La = {a, c, b, _2}
  imulq b, _2: La = {a, _2, c}
  movq $4, _3: La = {a, _2, _3, c}
  imulq a, _3: La = {_3, _2, c}
  movq _3, _4: La = {_2, _4, c}
  imulq c, _4: La = {_2, _4}
  movq _2, _5: La = {_4, _5}
  subq _4, _5: La = {_5}
  movq _5, %rdi: La = {%rdi}
  callq print_int: La = {}
```

For details see EoC, section 4.2.
However note that it recommends using a dictionary mapping instruction to live-sets.
This is fine if you use `support/x86_ast`, but if you roll your own, it may be better to use instruction numbers or (number,instruction) pairs as keys.

Compute the set of call-live variables (variables live during a call) - they should not be placed in caller-save registers.

2. Familiarise yourself with the `graph.py` helper module (or roll your own).
You can use graphviz for graph visualisation (not necessary for the compiler itself).


```
$ uv add graphviz
$ uv run python
>>> edges = [('a', 'b'), ('b','c'), ('c', 'd'), ('d', 'a')]
>>> g = UndirectedAdjList(edges)
>>> g.adjacent('a')
['b', 'd']
>>> gdot = g.show()
>>> print(gdot.source)
graph {
        a
        b
        c
        d
        b -- c [len=1.5]
        c -- d [len=1.5]
        a -- b [len=1.5]
        d -- a [len=1.5]
}

>>> gdot.render(filename='mygraph')
'mygraph.pdf'
>>> gdot.render(view=True)
'mygraph.pdf'
>>>
```

docs: https://graphviz.readthedocs.io/en/stable/manual.html

3. Write a function to build an interference graph based on live variable sets computed before.

Add the registers to the graph and precolor them with their respective colors;
add interference edges between call-live variables and caller-save registers.

For details see EoC, section 4.3.

Compare the computed graphs to the manually constructed ones.

4. Write a function to color the graph with natural numbers. For starters you can use a simple greedy algorithm:
always use the lowest color not used by the neighbours. Later you can refine it to use the saturation heuristic described in section 4.4 of EoC.
Note: vertex colors can be kept in a separate map; no need to store them in vertices themselves.


5. Modify `assign_homes` to use register allocation via graph coloring
- map low numbers to registers and when these are exhausted, map subsequent colors to stack locations.
Keep the version that used stack locations only for comparisons with the new one.

Create at least five programs that exercise all aspects of the register allocation algorithm,
including spilling variables to the stack (to this end, instead of using a zillion variables,  you can add a parameter telling how many registers to use)

6. Modify `prelude_and_conclusion` to save and restore used callee-save registers. Remember to keep stack aligned to 16 bytes.


Possible simplified (inefficient) initial approach:
- save/restore all callee-saved registers in prelude/conclusion
- allocate only callee-saved registers

## Final checks

Write some tests; make sure all key functionalities are covered.

- interfering variables get different registers
- noninterfering variables may get the same register
- spilling
- call-live variables get callee-saved registers
- used callee-saved registers are saved in the preluce and restored in the conclusion
- stack alignemnt

## Submission

- discuss with your tutor on the 4th (5p) or 5th (4p) lab
- submit to moodle before presenting
- submit a single `<uid>.tar.gz` file, where uid is your user id on students, in the format `xy128410`

After unpacking the archive, the compiler should be runnable with `uv run compiler.py <input file>`.
Document options (in particular limiting register count) in README.

## Recommended practices

Use git (or Jujutsu over git) for version management **from the start**.

Use uv for Python project management you can point it at support files instead of copying them:

```
[tool.uv.sources]
mrj-support = { path = "../support", editable = true }
```
