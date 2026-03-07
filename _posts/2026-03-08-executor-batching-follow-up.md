---
layout: writing
title: “Postgres: executor batching, follow up”
tags: [writing, pg]
last_updated: 2026-03-08
---
# Postgres: executor batching, follow up

March 8, 2026

About seven months ago I wrote a post sketching what executor batching in
Postgres might look like and what it might achieve. Since then I have been
actually implementing it, and the results are more interesting – and in places
more humbling – than the original post suggested. This is a follow-up on what
worked, what turned out to be harder than expected, and what the implementation
revealed that the perf profile alone did not.

## What worked: batching at the table AM boundary

The core idea from the original post – have the table AM return small batches
of tuples rather than one at a time, and process them in a loop at the scan
node – turned out to work roughly as expected. The patch series introduces a `RowBatch` API where the AM populates a batch
during the scan and the executor processes it in a loop before fetching the
next one. The batch is opaque to the executor – the AM is free to populate it
in whatever internal format suits it, and the executor accesses slots from the
batch without knowing how the AM has arranged the underlying tuples.

The reason the batch surfaces slots at all is that the rest of the executor
speaks slots and nothing else. The expression evaluator expects tuples in
slots, and every plan node above the scan communicates in slots. There is no
way to avoid them at the boundary without changing far more of the executor
than this patch series attempts. So the opaqueness is not an escape from slots
– it is freedom in how and when the AM materializes tuples into them, not
freedom from the abstraction itself. An AM that keeps tuples in a compact
internal format still has to hand a slot back when the callback fires; what it
avoids is being forced to fill every slot in the batch eagerly regardless of
how many the executor actually consumes.

This is worth dwelling on because the obvious naive implementation – just
pre-allocate an array of slots with `RowBatchCreateSlots()` and have the AM
fill them all upfront – is still fundamentally row-oriented. Each slot carries
its own `HeapTuple` pointer, its own deform state, its own null bitmap. You
have a loop, but you have not escaped the per-row overhead that slots embody.
The opaque-batch-with-callbacks design is what leaves room for AMs to do
something genuinely different underneath without the executor needing to know
about it.

The performance numbers are concrete: 10-42% improvement depending on the
workload, with the higher end showing up on in-memory analytical queries where
the executor loop is the dominant cost. That range is wide because the gains
are sensitive to the ratio of tuples that survive the qual filter – a
selective filter means fewer tuples reach the upper plan nodes and the batching
advantage is diluted, while a low-selectivity filter or no filter at all shows
the full benefit.

The amortization of function call overhead and the improvement in instruction
cache behavior are genuine. Running the same deforming code on consecutive
tuples in a tight loop really does behave better than the one-at-a-time path,
and the numbers confirm it.

## Where the profile was misleading

The original post pointed at `ExecInterpExpr()` consuming 71% of CPU and drew
the obvious conclusion: make it process tuples in batches and that 71% should
shrink. That conclusion is roughly correct, but the mechanism matters more than
the post implied.

The naive approach is to take the existing `ExprState`/`ExprEvalStep`
interpreter and call it in a loop over the batch. This helps somewhat, but when
we tried extending the interpreter’s opcode dispatch table with batch-specific
opcodes, a regression appeared in the batch=0 baseline case. The hypothesis is
that the larger dispatch table affects register pressure and instruction cache
behavior in `ExecInterpExpr()`’s main switch even for code paths that never
touch the new opcodes. But this is not a fully characterized finding – the
effect only showed up consistently under pgbench, which carries enough variance
that it is hard to say with confidence how much of it is real and how much is
noise. Designing around an effect we cannot cleanly reproduce is not a
satisfying position.

## A standalone batch qual evaluator

Whatever the status of that regression, there is a cleaner argument for a
standalone batch qual evaluator on its own merits: the `ExprState`/`ExprEvalStep`
infrastructure carries overhead that is simply unnecessary for the common case
of simple filter predicates over builtin types. The standalone evaluator handles
that case – comparisons involving builtin operators on a fixed set of supported
types – using a type-specialized fast-path comparison function table
(`BatchCmpFn`) that avoids fmgr dispatch entirely and does not touch
`ExprState` at all.

This is a different architectural conclusion than the original post gestured at.
The original framing was that batching was mainly about amortizing function call
overhead and improving cache locality within the existing evaluator. The more
useful framing is that for the restricted but practically important class of
quals that appear in most analytical queries, the evaluator infrastructure
itself is the overhead, and bypassing it entirely is the right design regardless
of what the dispatch table regression turns out to be.

The supported type set for the fast path covers the operators you most commonly
see in analytical filter conditions – integer comparisons, text equality, and a
handful of other builtins – which is enough to capture most real workloads even
if it is not general.

## What is still wishful thinking

The original post mentioned SIMD as a direction the batching work could
eventually enable. That remains true in principle but is further away than the
framing implied. The current batch evaluator is a tight loop with specialized
comparisons, which is a prerequisite for SIMD, but actually getting the
compiler to emit useful vector instructions requires more care than just
structuring the loop correctly – type widths, alignment, and the irregular
shape of variable-length types all get in the way. It is not impossible, but
the original post made it sound like a natural next step where in practice it
is a separate and nontrivial project.

The idea that a `TupleBatch` representation eliminating per-tuple metadata
redundancy would be a straightforward evolution from the slot array approach
also turned out to be more involved than expected. The slot array works and is
a reasonable intermediate point, but the step toward a genuinely compact batch
format touches enough of the AM interface and the executor node contract that it
is not a simple follow-on patch.

## Where things stand

The batched sequential scan infrastructure is in reasonable shape as a patch
series targeting Postgres 19. The table AM batch API, the `RowBatch` slot
management, and the standalone batch qual evaluator are all present. The
performance regression from extending the interpreter dispatch table is resolved
by keeping the batch evaluator separate.

The foreign key fast-path work I have also been developing in parallel showed
similar patterns: the SPI-based lookup that the original post described as the
mechanism for RI enforcement is where most of the overhead is in CPU-bound FK
workloads, and the gains from bypassing it are in the 2-3x range, which is
consistent with the general theme that per-row infrastructure overhead in the
current executor is larger than it looks in coarse profiles.

The original post’s overall framing – that batching is the right near-term
direction for executor improvement, sitting between the current row-at-a-time
design and a hypothetical vectorized executor – holds up. But the implementation
revealed that the interesting problem is not just batching the work; it is
identifying which parts of the existing infrastructure cannot simply be reused
in a batch loop and need to be rethought from a different angle.
