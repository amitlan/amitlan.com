---
layout: writing
title: "Postgres: how foreign keys work"
tags: [writing, pg]
last_updated: 2026-03-07
---
"Postgres: how foreign keys work"

Foreign key constraints are one of those features that most people who've used a relational database take for granted. You write `REFERENCES`, the database enforces it, and that's that. But the actual implementation inside Postgres is more interesting than it might seem, and I thought it'd be worth a writeup of what actually happens under the hood — from the objects created at DDL time to how enforcement works during DML.

## What gets created

When you issue an `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY` (or equivalently, a `REFERENCES` clause in `CREATE TABLE`), Postgres does a few distinct things.

The first thing is a row in `pg_constraint` with `contype = 'f'`. This stores the constraint's metadata: the referencing table (`conrelid`), the referenced table (`confrelid`), the arrays of column attribute numbers on both sides (`conkey` and `confkey`), and the configured match type and action codes for `ON UPDATE` and `ON DELETE` (`confmatchtype`, `confupdtype`, `confdeltype`). This catalog row is the durable record of the constraint; everything else is derived from it at enforcement time.

The second — and perhaps more surprising — thing is a set of triggers. Postgres implements RI enforcement entirely through triggers, both on the referencing table and on the referenced table. Specifically:

- On the **referencing table**, an `AFTER INSERT` trigger and an `AFTER UPDATE` trigger are created. Their job is to verify that a newly inserted or updated referencing row has a matching row in the referenced table.
- On the **referenced table**, an `AFTER DELETE` trigger and an `AFTER UPDATE` trigger are created. Their job is to respond when a referenced row goes away or its key changes — cascading the change, nullifying referencing rows, or raising an error, depending on the configured action.

These triggers are "internal" triggers — they are created with `tgisinternal = true` and are invisible to `\d` in psql by default, though they do show up in `pg_trigger`. Their trigger functions all live in `src/backend/utils/adt/ri_triggers.c`, which is the heart of the RI implementation. The specific function names follow a pattern like `RI_FKey_check_ins`, `RI_FKey_check_upd`, `RI_FKey_noaction_del`, `RI_FKey_cascade_del`, `RI_FKey_setnull_del`, and so on — one per action type per operation. The function to create these triggers when processing the DDL is `createForeignKeyActionTriggers()` in `src/backend/commands/tablecmds.c`, called from `ATAddForeignKeyConstraint()`.

One more thing that happens at `ADD CONSTRAINT` time: Postgres validates the existing data (unless you used `NOT VALID`). This is done by scanning the referencing table and checking each row against the referenced table. If any violations are found, the constraint addition fails. With `NOT VALID`, this scan is skipped and the constraint is marked accordingly in `pg_constraint.convalidated`; a subsequent `VALIDATE CONSTRAINT` does the scan separately, which has the advantage of only holding a weaker lock during that phase.

## How inserts and updates on the referencing side work

When a row is inserted into the referencing table, the `AFTER INSERT` row-level trigger fires, invoking `RI_FKey_check_ins()`. This function reads the constraint metadata from `pg_constraint` using the trigger's `tgconstraint` field, then uses SPI (the Server Programming Interface) to execute a query roughly of the form:

```sql
SELECT 1 FROM referenced_table WHERE ref_col1 = $1 AND ref_col2 = $2 ... LIMIT 1
```

using the newly inserted row's key values as parameters. If no matching row is found, it raises an error. The same logic applies for `RI_FKey_check_upd()` when the referencing key columns are modified by an `UPDATE`.

There's an optimization here: `ri_triggers.c` maintains a per-backend cache (`ri_query_cache`) that stores the SPI plan for a given constraint so it doesn't have to be prepared from scratch on every single invocation. The cache is keyed by constraint OID and the kind of check being performed.

One subtlety worth noting is the handling of NULLs. The `MATCH` type of the constraint affects what happens when key columns contain NULLs. With `MATCH SIMPLE` (the default), if *any* of the key columns is NULL, the check is skipped entirely — that is, a NULL key value is allowed without a corresponding referenced row. With `MATCH FULL`, *all* key columns must either be NULL or all non-NULL; a partial NULL is an error. With `MATCH PARTIAL`, the intent is that any non-NULL subset of the key must match, but `MATCH PARTIAL` is currently not implemented in Postgres and attempting to use it will raise an error.

## How deletes and updates on the referenced side work

When a row is deleted from the referenced table, the `AFTER DELETE` trigger fires. The specific function called depends on the `ON DELETE` action:

- `NO ACTION` (the default) and `RESTRICT` both call `RI_FKey_noaction_del()` and `RI_FKey_restrict_del()` respectively. Both check whether any referencing rows exist. The difference is timing: `RESTRICT` fires and checks immediately (it cannot be deferred), while `NO ACTION` can be deferred to end-of-transaction, allowing the referencing rows to be cleaned up within the same transaction before the check fires.
- `CASCADE` calls `RI_FKey_cascade_del()`, which deletes any referencing rows that point to the deleted referenced row.
- `SET NULL` calls `RI_FKey_setnull_del()`, which sets the referencing key columns to `NULL`.
- `SET DEFAULT` calls `RI_FKey_setdefault_del()`, which sets the referencing key columns to their column defaults.

The cascade and set-null/default variants again use SPI to execute the appropriate `DELETE` or `UPDATE` on the referencing table with the old key values. For `ON UPDATE`, the analogous functions exist for each action type.

The distinction between `NO ACTION` and `RESTRICT` is particularly interesting when constraints are declared `DEFERRABLE`. When deferred, the trigger is not fired immediately after each row operation — instead, it's queued and fired at end-of-transaction (or when `SET CONSTRAINTS ... IMMEDIATE` is explicitly called). This allows patterns like temporarily violating the constraint within a transaction so long as it's restored by commit time.

## A note on partitioned tables

Foreign keys interacting with partitioned tables have a few special considerations that are worth mentioning briefly.

When the **referencing table is partitioned**, a foreign key defined on it is replicated to each partition as an identical constraint. This is because the actual data lives in the partitions, and it's the partition-level triggers that will fire during DML. The constraint on the root partitioned table is mostly a logical record; enforcement happens at the leaf partition level.

When the **referenced table is partitioned**, Postgres does support this since version 12. The indexes needed for FK enforcement exist on the individual partitions, and the RI check functions know to scan through the partition hierarchy rather than treating the root as a monolithic table.

A special case arises during cross-partition `UPDATE`s on a partitioned table — when a row moves from one partition to another because the partition key is updated. This is internally implemented as a delete from the old partition followed by an insert into the new one. From the point of view of any foreign key constraint where this partitioned table is the **referenced** table, that delete could look like the referenced row is going away, which might incorrectly trigger the `ON DELETE` action. Postgres handles this with `ExecCrossPartitionUpdateForeignKey()` (in `src/backend/executor/execPartition.c`), which suppresses the delete-side RI triggers and instead fires the update-side triggers, so that `ON UPDATE` actions are applied correctly rather than `ON DELETE` ones. Without this, a cross-partition update of the referenced key would cascade-delete referencing rows rather than cascade-updating them, which would be wrong.

## Putting it together

So in summary: a foreign key constraint is represented by a `pg_constraint` row and a handful of system triggers, with all the enforcement logic in `ri_triggers.c`. The RI check itself is fundamentally a parameterized index lookup via SPI, with a per-backend plan cache to avoid repeated plan preparation overhead. The trigger-based approach is elegant in that it reuses Postgres' general trigger machinery, and the SPI-based checks mean the RI logic is relatively straightforward high-level C rather than needing to directly open indexes itself.

The flip side is that this architecture has a nontrivial overhead per row, especially when doing bulk loads or highly concurrent inserts into the referencing table. Each row insert generates a separate SPI query execution for the constraint check. That overhead is something I've been looking at more closely recently, but that's a topic for another post.

---

## Source files referenced:
- src/backend/utils/adt/ri_triggers.c
- src/backend/commands/tablecmds.c
- src/backend/catalog/pg_constraint.c
- src/backend/executor/execPartition.c
