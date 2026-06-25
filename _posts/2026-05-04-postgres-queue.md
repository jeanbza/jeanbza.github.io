---
layout: post
title:  "Queues in Postgres"
date:   2026-05-06 02:55:23 -0600
categories: 
toc: true
---

At Netflix, part of the infrastructure I work on is a distributed priority queue
built on Redis. It's great in many ways, but also pretty expensive and complex.

I'll detail that another time. For now, I'm going to focus on a much simpler way
to solve this if you have lesser constraints than what we do. This is based on
Postgres but would work for most relational databases. I've not load tested this
and couldn't really tell you how it scales, but it should be "good enough" for
most queueing under ~1k QPS and <1M messages (and probably fair amount higher,
but again, haven't load tested).

Note: This was used at our
[golang-index](https://github.com/Netflix-Skunkworks/golang-index) as well as in
[Go's pkgsite Postgres queue](https://github.com/golang/pkgsite/commit/372618454cdb62e4cbaab1fd14c58f2faf5db80a).

## Schema

```sql
-- This is the queue table.
CREATE TABLE IF NOT EXISTS queue_tasks (
    -- Bookkeeping.
    id          BIGSERIAL PRIMARY KEY,
    
    -- Each consumer/dequeue-er needs a unique task name. appID+threadID for example.
    task_name   TEXT UNIQUE NOT NULL,
    
    -- [...] Whatever payload you need for each message goes here.

    -- Queueing.
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at  TIMESTAMPTZ

    -- Optionally also add completed_at and maybe success/failure columns if you
    -- want a history; otherwise just delete on finish.
);
CREATE INDEX IF NOT EXISTS idx_queue_tasks_started_created
ON queue_tasks (started_at, created_at);
```

## Enqueueing

Enqueue with an atomic test-and-set query:

```sql
INSERT INTO queue_tasks (...) -- All but created_at.
VALUES (...)
ON CONFLICT (task_name) DO NOTHING; -- Another consumer won the race; no-op.
```

## Dequeueing

Dequeue with a test for stalled workers:

```sql
WITH next_task AS (
    SELECT id
    FROM queue_tasks
    WHERE started_at IS NULL
        -- Allow accepting work from stalled workers.
       OR started_at + INTERVAL '5 minutes' < NOW()
    ORDER BY created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE queue_tasks
SET started_at = NOW()
WHERE id = (SELECT id FROM next_task)
RETURNING id, started_at
```

And then to complete work, you'd simply `DELETE` the row.

### With priority

Simple add a `priority` field of your choosing and then `ORDER BY priority`
instead of `ORDER BY created_at`.

### With batch dequeue

To batch dequeue simply change `LIMIT 1` to `LIMIT n`.

### With lease extension

If you need lease extension, you can add `leased_until` and change the stall check
to:

```sql
WITH next_task AS (
    SELECT id
    FROM queue_tasks
    WHERE started_at IS NULL
        -- Allow accepting work from stalled workers; but now we used
        -- leased_until, which workers are expected to periodically lease
        -- extend.
       OR leased_until < NOW()
    ORDER BY created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE queue_tasks
SET started_at = NOW(),
    leased_until = NOW() + INTERVAL '5 minutes' -- Or however long you want leases.
WHERE id = (SELECT id FROM next_task)
RETURNING id, started_at
```

And then have your workers periodically update `leased_until` to some time in
the near future.

### With max consumers

If you need to limit work to a maximum amount of in-flight work/consumers, you can do:

```sql
WITH active_workers AS (
    SELECT COUNT(*) AS current_count
    FROM queue_tasks
    WHERE started_at IS NOT NULL
      AND started_at + INTERVAL '5 minutes' >= NOW()
),
next_task AS (
    SELECT id
    FROM queue_tasks
    WHERE (SELECT current_count FROM active_workers) < :n
      AND (started_at IS NULL OR started_at + INTERVAL '5 minutes' < NOW())
    ORDER BY created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE queue_tasks
SET started_at = NOW()
WHERE id = (SELECT id FROM next_task)
RETURNING id, started_at
```

## Conclusion

SQL databases are pretty easy to turn into queueing databases, and it's
fairly easy to tweak your SQL scripts to make the queue behave the way you want
them to.

If you already rely on a SQL database and need to add queueing, consider just
doing it in your pre-existing SQL database rather than adding a whole new system
dependency.
