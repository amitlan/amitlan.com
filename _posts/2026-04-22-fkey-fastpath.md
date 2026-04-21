---
layout: writing
title: "Postgres: faster foreign key checks"
tags: [writing, pg]
last_updated: 2026-04-22
---

# Postgres: faster foreign key checks
April 22, 2026

In an [earlier post](https://amitlan.com/2026/03/07/foreign-key-internals.html) I described how foreign key enforcement works in Postgres. The short version is that every INSERT or UPDATE on the referencing table fires an AFTER trigger that checks whether the new row’s FK column values exist in the referenced (PK) table. That check goes through SPI: it builds a query, plans it, executes it, and tears it all down, for every single row.

This is expensive. For a bulk INSERT of a million rows into a table with a foreign key, you’re running a million mini-queries against the PK table’s index. Each one opens the PK relation, acquires a snapshot, does a permission check, runs the index probe, and closes everything. The per-row cost is small in absolute terms, but it adds up fast.

For Postgres 19, I committed two patches (with Junwang Zhao as co-author on both) that bypass SPI entirely for the common case and batch the index probes. Together they make bulk FK inserts about 2.9x faster in the benchmark I used (int PK, int FK, 1M rows, PK table and index in memory).

## What the fast path does

The first patch ([2da86c1ef9](https://git.postgresql.org/pg/commitdiff/2da86c1ef9)) adds a fast path that replaces the SPI round-trip with a direct index scan. When the FK trigger fires, instead of building a query string and going through the full executor, `ri_FastPathCheck()` opens the PK relation and its index directly, sets up a scan key from the FK column values, and does the index probe in place. It handles the snapshot, permission checks, and security context switch the same way SPI would, just without the overhead of query parsing, planning, and executor setup.

The fast path applies when the PK table is not partitioned and the FK constraint uses simple equality on columns that have a suitable index. Partitioned PK tables still go through SPI, because routing the probe to the right partition would duplicate too much of the executor’s partition pruning logic.

This alone gives about a 1.8x speedup.

## Batching

The second patch ([b7b27eb41a](https://git.postgresql.org/pg/commitdiff/b7b27eb41a5cc0b45a1a9ce5c1cde5883d7bc358)) adds batching on top of the fast path. Instead of probing the index once per trigger invocation, it buffers FK rows in a per-constraint cache entry (`RI_FastPathEntry`) and flushes them in batches.

On each trigger invocation, `ri_FastPathBatchAdd()` copies the FK values into the buffer. When the buffer fills (64 rows) or the trigger-firing cycle ends, `ri_FastPathBatchFlush()` probes the index for all buffered rows at once. For single-column FKs, it builds an `ArrayType` from the buffered values and uses `SK_SEARCHARRAY` to scan the index in a single ordered traversal, rather than descending from the root once per row. The index AM sorts and deduplicates the array internally, then walks matching leaf pages in order. A `matched[]` bitmap tracks which batch items were satisfied; the first unmatched item is reported as a violation.

Multi-column FKs fall back to per-row probing within each flush, because there’s no equivalent of SK_SEARCHARRAY for composite keys.

The per-constraint cache entry also holds the open PK relation, index, scan descriptor, and tuple slot across trigger invocations within a batch, avoiding repeated open/close overhead. The snapshot and scan descriptor are taken fresh per flush.

This adds roughly another 1.6x on top of the first patch’s 1.8x.

## Lifecycle management

The trickiest part of the implementation is managing the lifetime of the cached resources. The `RI_FastPathEntry` has to survive across trigger invocations within a single trigger-firing cycle but must be torn down reliably when the cycle ends, including in error and abort paths.

The solution uses a new `AfterTriggerBatchCallback` mechanism in trigger.c. The RI code registers a callback (`ri_FastPathEndBatch`) that fires at the end of each trigger-firing cycle, whether that’s `AfterTriggerEndQuery` for immediate constraints, `AfterTriggerFireDeferred` at COMMIT, or `AfterTriggerSetState` for SET CONSTRAINTS IMMEDIATE. The callback flushes any partial batch and tears down cached resources.

For the abort path, an `XactCallback` and `SubXactCallback` null out the static cache pointer at transaction or subtransaction end, preventing the batch callback from accessing already-released resources.

There’s also a guard for the ALTER TABLE validation case. During `ALTER TABLE ... ADD FOREIGN KEY`, RI triggers are called directly, outside the after-trigger framework, so batch callbacks would never fire. The fast-path code checks `AfterTriggerBatchIsActive()` and falls back to the non-cached per-invocation path when it returns false.

## The snapshot question

One design decision worth mentioning is the snapshot handling. The SPI path takes a fresh snapshot per row via `GetTransactionSnapshot()`. The fast path takes one snapshot per flush (per 64 rows). Under READ COMMITTED, this means the fast path won’t see PK rows committed by other backends between flushes. But that visibility is non-deterministic even in the SPI path, since whether a given row’s check happens to see a concurrent commit is a race. The FK check only needs PK rows visible before the statement began plus effects of earlier triggers (tracked by curcid), and `LockTupleKeyShare` prevents the PK row from disappearing regardless.

## Credits

David Rowley suggested the SK_SEARCHARRAY approach for batching, which is what makes the second patch’s speedup possible. Junwang Zhao co-authored both patches and did the initial implementation of the fast path. Haibo Yan, Chao Li, and Tomas Vondra provided reviews and testing.

The thread where this was developed: [pgsql-hackers](https://postgr.es/m/CA+HiwqF4C0ws3cO+z5cLkPuvwnAwkSp7sfvgGj3yQ=Li6KNMqA@mail.gmail.com).
