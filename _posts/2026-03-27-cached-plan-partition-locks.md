---
layout: writing
title: "Postgres: cached plan partition locking bottleneck"
tags: [writing, pg]
last_updated: 2026-03-27
---

# Postgres: cached plan partition locking bottleneck

March 27, 2026
 
In two earlier posts ([one](https://amitlan.com/2022/05/16/param-query-partition-woes.html), [two](https://amitlan.com/2022/11/24/query-perm-check.html)), I described performance problems that arise when using prepared statements with partitioned tables. The core issue is that when Postgres reuses a cached generic plan, it locks every partition mentioned in the plan upfront, even those that run-time pruning will immediately eliminate. For a table with thousands of partitions where a typical query touches only one, this means thousands of unnecessary `LockRelationOid()` calls on every execution.
 
I proposed a patch back in 2022 to address this. The idea was straightforward: perform initial pruning before acquiring execution locks, then lock only the partitions that survived. After a long review cycle, the patch was [committed](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=525392d5727f469e9a5882e1d728917a4be56147) for Postgres 18 in February 2025. Three months later, Tom Lane found a serious flaw in how it interacted with plan invalidation and the commit was [reverted](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1722d5eb05d8e5d2e064cd1798abcae4f296ca9d).
 
This post is about what went wrong, what the redesign looks like, and where things stand now.
 
## What went wrong
 
The original approach deferred partition locking until after initial pruning ran inside `ExecutorStart()`. That part worked. The trouble was what happened when the plan got invalidated during execution, because a partition was altered in the window between validation and locking. The patch needed to replan, and it did so from inside `ExecutorStart()`, replacing the `PlannedStmt`s in `CachedPlan->stmt_list` with freshly planned ones. To avoid having to retract all the way back to the caller running the loop over `stmt_list` (in portal, SPI, or EXPLAIN code), it reused the same underlying `List` structure so those outer loops could continue iterating. The new plans were allocated in separate memory, but the in-place replacement of the list contents was the core trick.
 
Tom Lane [found](https://postgr.es/m/605328.1747710381@sss.pgh.pa.us) that this broke when rule rewriting changed the number of statements between the original plan and the replan. The list length changed, but the outer loop didn't know. He also observed that the whole premise was fragile: an outer caller could still be mid-execution of an earlier statement in the list when an inner call triggered replanning, and there was no safe way to reconcile the two.
 
Beyond the correctness issues, Tom objected to the API changes the patch introduced. `ExecutorStart()` callers had to know about the deferred locking machinery, which added complexity to code paths that had nothing to do with partitioning.
 
## The redesign
 
The new approach, which I've been working on for the past year, inverts the direction. Instead of deferring locking until the executor, it brings a small amount of executor work forward into the plan cache validation step.
 
When `GetCachedPlan()` validates a cached generic plan, it normally calls `AcquireExecutorLocks()` to lock every relation in the plan. The patch adds an alternative path, `AcquireExecutorLocksUnpruned()`, that does this instead:
 
1. Lock the unprunable relations (the partitioned parent tables and non-partition RTEs) from `PlannedStmt.unprunableRelids`.
2. Call a new function, `ExecutorPrep()`, which sets up the range table, runs permission checks, and evaluates the initial pruning expressions to determine which partitions survive.
3. Lock only the surviving leaf partitions.
 
`ExecutorPrep()` is factored out of what `InitPlan()` does today inside `ExecutorStart()`. It performs range table initialization, permission checks, and initial pruning. No plan tree initialization, no node init, no relation opening beyond the parent tables needed for partition descriptors (which are already locked in step 1). The resulting `EState` is then passed through to `ExecutorStart()`, which reuses it instead of rebuilding it from scratch.
 
The `CachedPlan` itself is never modified. All per-execution state flows through a new struct, `CachedPlanPrepData`, that the caller provides when calling `GetCachedPlan()`. The callers that care about pruning-aware locking (portals, SPI, EXECUTE, EXPLAIN) fill in this struct and thread the resulting `EState` to `ExecutorStart()` via `QueryDesc`. Callers that don't care pass NULL and get the old lock-everything behavior unchanged.
 
## The boundary crossing
 
The architecturally interesting (or uncomfortable, depending on your perspective) part is that `ExecutorPrep()` is executor code being called from `plancache.c`. The plan cache and executor have always had a clean boundary: the plan cache returns a valid plan with all needed locks held, and the executor runs it. This patch crosses that line.
 
The reason is fundamental: the information needed to decide which locks to skip (pruning results) can only come from executor machinery. The pruning expressions live in the plan tree and require an `EState` with range table initialization to evaluate. There is no way to compute the correct lock set from the `PlannedStmt` alone.
 
Tom Lane himself observed this tension back in January 2023, when he [suggested](https://www.postgresql.org/message-id/4191508.1674157166%40sss.pgh.pa.us) getting rid of `AcquireExecutorLocks()` entirely and integrating lock acquisition into executor startup, with the executor able to signal back that the plan is stale. The reverted approach attempted something in that spirit but without the clean retry mechanism, resorting instead to in-place replanning. `ExecutorPrep()` goes the other way: rather than moving locking into the executor, it moves just enough executor work into the plan cache to compute the correct lock set.
 
## Multi-statement plans
 
One case where pruning-aware locking is explicitly disabled is multi-statement `CachedPlan`s, which arise from rule rewriting. As described above, `PortalRunMulti()` issues `CommandCounterIncrement()` between statements, so later statements' pruning expressions can legitimately see different data than they would if evaluated upfront. There is also a memory lifetime issue: `PortalRunMulti()` calls `MemoryContextDeleteChildren(portalContext)` between statements, which would destroy `EState`s prepared for later statements.
 
The fix is simple: if the `CachedPlan` contains more than one `PlannedStmt`, fall back to locking all partitions. Single-statement plans, which cover the vast majority of prepared statements, are unaffected.
 
## Safety nets
 
The patch includes several debug-build checks to catch contract violations:
 
In `standard_ExecutorStart()`, when no prep `EState` was provided, an Assert verifies that every lockable relation in the plan's range table is actually locked. When a prep `EState` was provided, a parallel Assert checks that at least the unpruned relations are locked. Both are skipped in parallel workers, which acquire relation locks lazily in `ExecGetRangeTableRelation()`.
 
In `SPI_cursor_open_internal()`, an Assert verifies that `prep_estate` is NULL in the `!plan->saved` path, since unsaved plans always use custom plans and the subsequent `copyObject`/`ReleaseCachedPlan` sequence would leave the `EState` pointing at freed memory.
 
If the plan is invalidated between `ExecutorPrep()` and `ExecutorStart()` (for example, because a pruning expression triggered DDL on a partition), `CachedPlanPrepCleanup()` frees the orphaned `EState` and `GetCachedPlan()` retries. A regression test exercises this path using a stable SQL function wrapping a volatile PL/pgSQL function that performs DDL when signaled.
 
## What's in the patch
 
The series is structured as follows:
 
0001 refactors the executor's initial pruning setup: simplifies unpruned relid tracking and moves `ecxt_param_exec_vals` setup to allow pruning state creation before PARAM_EXEC parameters are available. Pure cleanup, no behavioral change.
 
0002 introduces `ExecutorPrep()` and refactors `ExecutorStart()` to reuse its `EState` when one is provided. Adds the scaffolding for carrying a prep `EState` through `CreateQueryDesc`, `PortalDefineQuery`, and SPI. All callers pass NULL at this point.
 
0003 is the core patch. It wires `GetCachedPlan()` into the pruning-aware locking path via `AcquireExecutorLocksUnpruned()`, introduces `CachedPlanPrepData`, adjusts all call sites, adds the multi-statement guard, and includes regression tests for locking behavior, `firstResultRels` with writable CTEs, plan invalidation during pruning, and multi-statement fallback.
 
Two additional patches extend the machinery to SQL functions (0004) and parallel workers (0005), but are independent of the core optimization and can be reviewed separately.
 
## What's next
 
The patch is under review for Postgres 19. Whether it makes the cut depends on getting sign-off from senior hackers on the architectural trade-off: is it acceptable for plan cache code to call executor code in order to compute the correct lock set? I think it is, because the alternative is either living with the O(n) locking bottleneck or redesigning the plan cache / executor boundary from scratch. But it is a judgment call, and reasonable people can disagree.
 
For users who rely on prepared statements with large partitioned tables, this would be a meaningful improvement. This particular instance of locking bottleneck has existed since declarative partitioning was added in Postgres 10 while some were fixed in Postgres 11 and 12, and while `plan_cache_mode = force_custom_plan` is a workaround, it trades away the whole point of plan caching.
