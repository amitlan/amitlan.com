---
layout: writing
title: "Postgres: one thread, one project"
tags: [writing, pg]
last_updated: 2026-04-12
---

# Postgres: one thread, one project
April 12, 2026

I've been working on the pruning-aware partition locking patch off
and on since 2022. It was committed to Postgres 18, reverted, and is
now on its second redesign targeting Postgres 20. Along the way I started having some thoughts about -hackers threads that I wish I have had a bit earlier: a thread should carry one project, not several.

The original thread started as "optimize generic plan execution for
partitions pruned by initial pruning" and stayed alive through three
distinct designs. Each time the design pivoted, the thread kept its
old subject and its old archive. Reviewers who had checked out the
thread a year earlier had no reason to check back. New reviewers
looking for an entry point found a 200-message archive with
conflicting proposals and no clear starting point.

The current patch is a clean architectural refactor followed by an
optimization. They share a motivation but they're separable -- the
refactor (move execution lock acquisition out of GetCachedPlan) is
independently useful and independently reviewable. Carrying both in
the original thread would have been the default choice. Instead,
I'm starting a new thread for the refactor alone, and a separate
thread for the optimization once the refactor lands.

The cost is duplication: both threads will cite the same motivation,
reference each other, maybe restate the same context. The benefit
is that each has a subject line that describes one thing, an archive
that tells one story, and a question that can be answered yes or no
without paging through a year of prior debate.

I suspect this is what more experienced committers do by default --
they don't let their threads carry more than one decision at a time.
I've been doing the opposite -- letting one thread accumulate every
design I considered along the way, under a subject line that stopped
describing the current state months ago -- and then wondering why
reviewers weren't showing up. The answer, in retrospect, is that
I was asking them to do archaeology before they could do review.

Some will disagree that splitting the thread is the right call, and
I'm not certain either. I just want to try something different.
I've reset threads on other projects in the past when they got too
tangled; for whatever reason, it didn't occur to me to do it here
until now. Time to find out if it helps.
