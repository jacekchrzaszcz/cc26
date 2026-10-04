## Lab 5 - loops and liveness across blocks

Goal: finish *Lif* by making liveness and register allocation reach across
a control-flow graph instead of falling back to the stack, then extend the
compiler to *Lwhile*, adding `while`.

The previous lab's compiler put every
variable in a stack slot rather than trying to run the liveness
analysis across a jump. This lab removes that fallback, and then adds
`while`, whose control-flow graph can contain cycles.

## Cross-block liveness

### The control-flow graph

Build a directed graph with one vertex per block (per label) and one edge
per jump: a plain `goto L` contributes one edge, a conditional tail
`if ... goto Lt else goto Lf` contributes two. (`graph.py` from lab 3 has
`DirectedAdjList`, `topological_sort` and `transpose` if you don't already
have your own.)

Some jump targets — the block that holds the function epilogue — have no
block of their own in the program; treat a missing block as an empty
instruction list, and hence an empty live-before set.

### What a jump contributes to liveness

Within a block, liveness runs backwards exactly as in lecture 3:
`live_before(k) = (live_after(k) - write(k)) ∪ read(k)`. Jumps read and
write nothing directly; what they contribute is their own **live-after**
set, taken from the live-**before** set of whatever block they target:

```
live_after(jmp L)    = live_before[L]
live_after(j<cc> L)  = live_before[L]  ∪  live_before(next instruction)
```

The union in the second line is not optional padding. Last lab's
`select_instructions` always emits a conditional jump immediately followed
by an unconditional one — `j<cc> Ltrue` then `jmp Lfalse` — so at the
`j<cc>` instruction, "the next instruction" is the `jmp Lfalse`, and its
own live-after is `live_before[Lfalse]`. Leave out the union and a
variable that's only live on the not-taken branch looks dead across the
test; it can end up sharing a register with something the taken branch is
still using.


### Acyclic first: process blocks in the right order

If a block's liveness depends on its successors' live-before sets, compute
the successors first. For a program built only from `if` (no loops yet),
the control-flow graph has no cycle, so such an order exists: it's a
topological order of the graph with edges reversed, i.e. process the
blocks in topological order of the **transposed** CFG. One pass over the
blocks in that order is then enough — by the time you reach a block, every
block it jumps to has already been processed.

If you feed this a graph that turns out to be cyclic (which will happen
the moment you write a `while`-containing test program below), a
topological sort has nowhere to put the blocks in the cycle: it's a good,
loud way to detect that your program has left `if`-only territory.

### Register allocation, for real

Once you have live-before/live-after sets for the whole program, feed the
whole program's instructions — every block's, not one block's — through
last lab's interference-graph and greedy-colouring code unchanged; it
doesn't know or care that the instructions came from different blocks. The
"more than one block ⟹ everything on the stack" special case from last lab
is now dead code — delete it.

**Checkpoint**: an `if`/`else` program with enough variables to force
spilling should now get real registers on both branches, not stack slots,
and a variable only used on one branch should not block a register that's
free on the other.

## Loops: *Lwhile*

Concrete syntax, on top of `Lif`:

```
stmt ::= ... | while exp: stmt+
```

Abstract syntax:

```
While(test, [stmt], [])
```

(`ast.While` carries a third field for Python's `while ... else ...`; we
don't support it — reject anything with a non-empty `orelse` rather than
silently dropping it.)

Add `while` to the interpreter, and to `remove_complex_operands` and
`explicate_control` as below.

### remove_complex_operands: the test runs more than once

For `if`, a temporary needed by the test can simply be hoisted as a
statement in front of the `If` — the test runs exactly once, wherever
control reaches it. A `while`'s test runs **every iteration**, so its
temporaries cannot be hoisted in front of the loop; they have to be
recomputed as part of the test, every time round.

```
while input_int() < 10:
    ...
```

The call needs a temporary, but that temporary must be re-evaluated on
every pass through the loop, so it stays *inside* the test rather than in
front of the `while`:

```
while (_1 = input_int(); _1 < 10):
    ...
```

using whatever "run these statements, then use this expression" construct
your intermediate representation already has for a branch's own
temporaries (the one `remove_complex_operands` introduced last lab to keep
a branch's temporaries from being evaluated on the path that doesn't take
that branch — the two problems are the same shape: something that must
run again, rather than being hoisted once).

### explicate_control: a loop is a header block with a back edge

No new tail form is needed. A `while` compiles to a block — call it the
*header* — that evaluates the test and branches: to the loop body on
true, to whatever follows the loop on false. The body, compiled as
statements, ends not in the loop's continuation but in a jump **back** to
the header.

That back edge is the one place this pass has to work in an order other
than "compile the pieces, then assemble them": the header's label has to
exist *before* you compile the body, because the body's last instruction
needs to jump to it. Generate the label first, fill in the block after:

```python
loop_label = fresh_label()
after_loop = create_block(cont, basic_blocks)        # what follows the loop
body_code  = explicate_stmts(body, [Goto(loop_label)], basic_blocks)
basic_blocks[loop_label] = explicate_pred(test, body_code, after_loop,
                                           basic_blocks)
return [Goto(loop_label)]
```

Reuse whatever `explicate_pred` already does with a `Compare` test; a
`while` test that turned into a `Begin` (see above) needs a case there too
— run the statements, then explicate the result as the condition.

`select_instructions` needs no new case at all: the header compiles
through the same "comparison as a block-ending tail" case from last lab,
and the body's back edge is just another `goto`.

## The control-flow graph now has a cycle

The reverse-topological order from earlier in this lab doesn't exist once
there's a back edge — there is no order in which every block's successors
come first, because the header is its own (indirect) successor. Liveness
across a cyclic graph needs a different algorithm: iterate to a fixpoint.

- Start **every** block's live-before set at `∅`. This is an
  underestimate, and that's fine — liveness only ever grows as you
  discover more of the graph, it never shrinks, so starting from nothing
  is a safe place to start.
- Process a block: recompute its live-before set from its successors'
  *current* (possibly still-wrong) live-before sets, exactly as in the
  acyclic case.
- If that recomputation **changed** the block's live-before set, every
  block that jumps to it might now compute something different too — put
  those predecessors back on the worklist.
- Stop when the worklist is empty.

This terminates because a live-before set only ever grows (never shrinks)
and the set of variables in the program is finite, so there's a limit to
how many times any block's set can change.

```python
worklist = deque(all_labels)
while worklist:
    label = worklist.pop()
    live_before = recompute(label)          # using each successor's *current* set
    if live_before != live_before_block[label]:
        live_before_block[label] = live_before
        worklist.extend(predecessors_of(label))   # via the transposed CFG
```

Note this algorithm also works on the acyclic case — the reverse-topological
sweep is really just the special case where every block's set stabilises
on the first visit, so you may prefer to implement only the fixpoint
version and drop the topological-order one entirely once it's working.

**Checkpoint**: a loop whose body contains a `print`/`input_int` call
should push and restore whatever callee-saved registers the colouring
used for variables that are live across that call, and a loop with enough
live variables should still spill correctly rather than crashing or
silently reusing a register.

### Stretch goals

- extend the interference-graph "move bias" from lab 3 (don't add an
  edge between a `movq`'s source and destination) to notice when doing so
  lets a loop-carried variable keep the same register across iterations.
- add `break`/`continue`: both need to know which block to jump to, which
  means threading the enclosing loop's header and exit labels through
  `explicate_stmt` for the duration of the loop's body.

## Final check
Write some tests; make sure all key functionalities are covered.

- short circuit evaluation
- if expressions and statements
- logical expressions in assignments and conditions
- temporary variables in branches and loop conditions do not escape
- blocks are not duplicated for nested conditionals
- nested loops
- liveness and register allocation with loops

## Submission

- discuss with your tutor on the 6th (5p) or 7th (4p) lab
- submit to moodle before presenting
- submit a single `<uid>.tar.gz` file, where uid is your user id on students, in the format `xy128410`
- include a README documenting nonobvious elements

After unpacking the archive, the compiler should be runnable with `uv run compiler.py <input file>`.

## Recommended practices

Use git (or Jujutsu over git) for version management **from the start**.

Use uv for Python project management you can point it at support files instead of copying them:

```
[tool.uv.sources]
mrj-support = { path = "../support", editable = true }
```
