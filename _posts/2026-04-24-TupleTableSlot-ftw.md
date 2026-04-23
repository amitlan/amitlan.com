---
layout: writing
title: "Postgres: TupleTableSlot as a batch container"
tags: [writing, pg]
last_updated: 2026-04-23
---

# Postgres: TupleTableSlot as a batch container”
April 23, 2026

I’ve been working on adding batched execution to the Postgres executor. In an
[earlier post](https://amitlan.com/2025/07/08/executor.html) I described where the
executor spends its time and why batching at the scan level is a natural first step, and
in a [follow-up](https://amitlan.com/2026/03/08/executor-batching-follow-up.html) I
wrote about the `RowBatch` API that came out of the initial implementation — what worked,
what didn’t, and what the implementation revealed. One thing that became clear was that
introducing a separate `RowBatch` container meant a parallel data path alongside the
existing slot machinery. After living with that for a while, I realized TupleTableSlot
might already have what’s needed — and adding a batch-aware slot type would avoid the new
module entirely. To explain why, it helps to look at what TupleTableSlot actually does
and how the executor uses it today. (The code blocks in this post are simplified for illustration; they
are not meant to be exact representations of the current or proposed code.)

Every tuple that flows through the executor passes through a TupleTableSlot. It’s the
interface the executor uses to access tuple data without knowing how the tuple is physically
stored. Here’s the base struct:

```c
typedef struct TupleTableSlot
{
    NodeTag          type;
    uint16           tts_flags;
    AttrNumber       tts_nvalid;
    const TupleTableSlotOps *tts_ops;
    TupleDesc        tts_tupleDescriptor;
    Datum           *tts_values;
    bool            *tts_isnull;
    MemoryContext    tts_mcxt;
    ItemPointerData  tts_tid;
} TupleTableSlot;
```

`tts_ops` is a vtable — a struct of function pointers that defines how to interact with
whatever concrete tuple representation is behind the slot. The executor never touches
tuple data directly. It calls `slot_getsomeattrs(slot, N)` to deform attributes into
`tts_values[]` and `tts_isnull[]`, and then reads the Datums from there. The expression
evaluator, qual evaluation, projection — all of them go through this path.

There are three concrete slot types (four if you count a subtype). `HeapTupleTableSlot`
holds a pointer to a HeapTuple:

```c
typedef struct HeapTupleTableSlot
{
    TupleTableSlot base;
    HeapTuple       tuple;
    uint32          off;
    HeapTupleData   tupdata;
} HeapTupleTableSlot;
```

`BufferHeapTupleTableSlot` extends it for tuples that live in a pinned buffer page:

```c
typedef struct BufferHeapTupleTableSlot
{
    HeapTupleTableSlot base;
    Buffer      buffer;
} BufferHeapTupleTableSlot;
```

`MinimalTupleTableSlot` holds a MinimalTuple — a stripped-down format without system columns,
used for intermediate results inside joins and sorts. `VirtualTupleTableSlot` has no physical
tuple at all — it’s just the `tts_values`/`tts_isnull` arrays directly, which is what projection
produces.

Each of these implements the same `TupleTableSlotOps` callbacks:

```c
struct TupleTableSlotOps
{
    size_t           base_slot_size;
    void (*init)(TupleTableSlot *slot);
    void (*release)(TupleTableSlot *slot);
    void (*clear)(TupleTableSlot *slot);
    void (*getsomeattrs)(TupleTableSlot *slot, int natts);
    Datum (*getsysattr)(TupleTableSlot *slot, int attnum, bool *isnull);
    void (*materialize)(TupleTableSlot *slot);
    void (*copyslot)(TupleTableSlot *dstslot, TupleTableSlot *srcslot);
    /* ... */
};
```

The `getsomeattrs` callback is the important one. For a heap slot, it calls
`heap_deform_tuple` to unpack attributes from the on-disk format. For a virtual slot,
it’s a no-op because the values are already there. The executor doesn’t know or care
which one it’s talking to.

Now here’s how a sequential scan uses this today:

```c
heapam_scan_getnextslot(TableScanDesc sscan,
                        ScanDirection direction,
                        TupleTableSlot *slot)
{
    HeapScanDesc scan = (HeapScanDesc) sscan;

    heapgettup_pagemode(scan, direction, ...);

    if (scan->rs_ctup.t_data == NULL)
    {
        ExecClearTuple(slot);
        return false;
    }

    ExecStoreBufferHeapTuple(&scan->rs_ctup, slot, scan->rs_cbuf);
    return true;
}
```

One call, one tuple. The AM fills a `HeapTupleData`, stores it in the slot, returns. The
executor evaluates quals, projects, returns the tuple up the plan tree, and calls back
into the AM for the next one. That per-tuple round trip is the overhead that batching
aims to reduce.

The thing is, the slot abstraction already separates what the executor needs (Datums,
null flags) from how the storage provides them. The executor goes through `getsomeattrs`,
reads `tts_values[]`, and that’s it. It never looks at the physical tuple. So if the slot
held a batch of tuples instead of one, and advanced through them internally, the executor
wouldn’t need to know.

That’s the idea behind adding a `BatchHeapTupleTableSlot`:

```c
typedef struct BatchHeapTupleTableSlot
{
    BufferHeapTupleTableSlot base;
    HeapTupleData  *tuples;
    int             ntuples;
    int             cursor;
} BatchHeapTupleTableSlot;
```

`tuples` points into the scan descriptor’s `rs_vistuples[]` array. Today that array holds
`OffsetNumber` entries — just the offset of each visible tuple on the page. But the heap AM
was already building full `HeapTupleData` headers for every visible tuple during visibility
checking in `page_collect_tuples()`, then throwing them away and saving only the offset. One
of the patches in this series changes `rs_vistuples[]` from `OffsetNumber[]` to
`HeapTupleData[]`, retaining the work that was already being done (see the
[“Batching in executor” thread](https://www.postgresql.org/message-id/CA%2BHiwqGkway9dcpTVmRs0Q%3DZsQQ8Nx0MRjy9jfDmR8QFLnd-Mg%40mail.gmail.com)
on pgsql-hackers). The page is the natural batch unit for heap, and the data was already
there — it just wasn’t being kept. No copy is needed here either, since `t_data` points into
the pinned buffer page.

The `getsomeattrs` callback for this slot type deforms whichever tuple the cursor currently
points at. Same deform code path, same `tts_values[]` output. The expression evaluator sees
no difference. Advancing through the batch is a function that knows the concrete slot type
and manipulates its fields directly, in the same way `ExecStoreBufferHeapTuple` does for
regular heap slots:

```c
static inline bool
ExecBatchSlotAdvance(TupleTableSlot *slot)
{
    BatchHeapTupleTableSlot *bslot = (BatchHeapTupleTableSlot *) slot;
    HeapTupleTableSlot *hslot = &bslot->base.base;

    if (bslot->cursor >= bslot->ntuples)
        return false;

    hslot->tupdata = bslot->tuples[bslot->cursor];
    hslot->tuple = &hslot->tupdata;
    bslot->cursor++;

    slot->tts_nvalid = 0;
    slot->tts_flags &= ~TTS_FLAG_EMPTY;

    return true;
}
```

The fetch function that `ExecScanExtended()` calls (inlined into variants like
`ExecSeqScanWithQual` to eliminate branches for qual and projection) changes from
“call the AM per tuple” to “call the AM per page, then advance locally”:

```c
static TupleTableSlot *
SeqNextBatch(SeqScanState *node)
{
    TupleTableSlot *slot = node->ss.ss_ScanTupleSlot;

    if (ExecBatchSlotAdvance(slot))
        return slot;

    ExecClearTuple(slot);

    if (!table_scan_getnextbatch(node->ss.ss_currentScanDesc,
                                 ForwardScanDirection,
                                 slot))
        return slot;

    ExecBatchSlotAdvance(slot);
    return slot;
}
```

`ExecScanExtended` itself is untouched. It calls the fetch function, gets a slot, evaluates
the qual, projects — same as before. The only difference is that `SeqNextBatch` calls
into the AM once per page (via `table_scan_getnextbatch`) instead of once per tuple. The
rest of the time it just bumps a cursor in the slot.

The slot is what makes this work with minimal new API surface. The only new tableam
callback is `scan_getnextbatch`. The AM fills a batch in whatever representation makes
sense for it — heap fills a `HeapTupleData[]` via `BatchHeapTupleTableSlot`, but a columnar
TAM would define its own batch slot type with entirely different internals (column arrays,
perhaps) and its own `getsomeattrs`. The executor doesn’t care — it sees `TupleTableSlot *`
and calls the same callbacks either way. That’s how the existing slot abstraction already
works; batching just adds one more slot type on the same pattern.

The tuple-at-a-time contract doesn’t live in the slot — it lives in how the executor
iterates and how often it calls the AM. The slot just holds whatever it’s given. Give it
one tuple, it holds one tuple. Give it a page’s worth, it holds a page’s worth. The
`getsomeattrs` callback handles the translation either way.

The other thing this establishes is a clean boundary for follow-up work on vectorized qual
evaluation. Once the batch slot carries the tuples, a single additional callback —
`slot_getsomeattrs_batch` — can produce per-column arrays (`Datum *values[attnum][ntuples]`)
from whatever the batch holds. For a heap batch slot, that means deforming every tuple in
the batch into columnar form — real work, but done once per batch rather than per tuple. For
a columnar TAM whose `getnextbatch` already populates native column arrays, the same callback
can be a no-op — the data is already in the right shape. Either way, a vectorized qual
evaluator consumes the same column arrays, and the AM never needs to know it exists. Its job
ends at `getnextbatch()`. That boundary is what makes the vectorized path possible without a
second round of AM changes.

Looking further ahead, the same slot-based approach extends to passing batches between
executor nodes — not just between the AM and the scan node. I had initially wondered whether
we’d need a separate `ExecProcNodeBatch` callback on `PlanState`, so a parent node could call
it instead of the per-tuple `ExecProcNode`. But if the batch lives in the slot rather than in
a separate container, a parent node can simply inspect the result slot of its child to determine
whether it holds a batch and engage batch-aware logic accordingly. Since all slots are created
during `ExecInit`, each parent can check its child’s result slot type at init time and select a
batch-aware execution variant — the same pattern used to select `ExecSeqScanBatchWithQual` vs
`ExecSeqScanWithQual`. By the time execution starts, the entire tree is wired. A Hash Agg node,
for example, could see that its child’s result slot is a batch slot and hash the entire batch in
one call instead of pulling one tuple at a time. And because a batch slot is still a
`TupleTableSlot`, a parent that hasn’t been taught batch consumption yet just pulls tuples one at
a time through the existing callbacks — the batch property is there but unused. Each node opts in
independently, at its own pace, with no breakage. The gating piece for this is batch-aware
projection: for a node to output batches in its result slot, the targetlist evaluation for
qualifying tuples must itself be batch-aware. Until that exists, the batch collapses back to
tuple-at-a-time at each node boundary. That’s probably v20 or v21 territory, but nothing about
the current design closes the door.
