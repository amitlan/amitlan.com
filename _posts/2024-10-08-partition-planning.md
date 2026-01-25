---
layout: writing
title: "Postgres: planning with partitions"
tags: [writing, pg]
last_updated: 2024-10-08
---
# Postgres: planning with partitions

Oct 8, 2024

I've been meaning to write about this topic for a while so here it is.  The main goal is
to explain how the number of partitions a table has affects the planning time of a query
referencing only that table, both when the query's `WHERE` condition can be used to
perform [partition pruning](https://www.postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITION-PRUNING)
and when it can't.  I will mention the names of some functions in the Postgres source
code that are involved in the planning of such queries so that anyone who is interested
to further explore can jump into the source code to understand this better.

Let's get right into it.  We'll use the following partitioned table:

```
create table foo (a int) partition by range (a);
create table foo_1 partition of foo for values from (1, 1000);
create table foo_2 partition of foo for values from (1000, 2000);
...
create table foo_N partition of foo for values from ((N - 1) * 1000, N * 1000);
```

When a user sends a query like this:

```
select * from foo where a = 1;
```

Partitions `foo_1` to `foo_N` are not looked at until the query reaches
the planner function `query_planner()`.  `query_planner()` is a function that is
responsible for deciding how to scan the individual tables and also how join if
there is more than one table in the query.
