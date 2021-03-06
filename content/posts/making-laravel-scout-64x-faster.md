---
title: Making Laravel Scout Imports 64x Faster
date: 2019-03-11T12:00:00-05:00
description:
images: ["img/scout-imports.png"]
tags: ["laravel", "php"]
---

The next release of [Laravel Scout](https://laravel.com/docs/5.8/scout) will be able to index your models ~64x faster [^1].  You can see the pull request [here](https://github.com/laravel/scout/pull/360).

This post explains why the change makes such a huge difference.  This is a technique I've used in my own Laravel applications to speed up problematic queries.  Once you understand how it works I bet you will find a few queries in your own application to optimize.

## Background

By default Scout will index every row in your database. To keep memory usage reasonable it only fetches a few hundred rows at a time using the [`chunk` method](https://laravel.com/docs/5.8/queries#chunking-results).

The `chunk` method adds a limit and an offset to your query, like this:

```sql
select * from `users` order by `id` asc limit 500 offset 500;
```

To fetch the next page the offset is incremented.

```sql
select * from `users` order by `id` asc limit 500 offset 1000;
```

This works well at first - this query only takes about a millisecond - but it gets slower the farther you go.

Once you get to offset 1,000,000 the query can easily be 500 times slower.  When a Scout import executes thousands of these queries, it adds up.

## Why Offset is Slow

When you use offset you're telling the database to skip the first N rows. Even though the skipped rows are not returned, the database still has to read them from disk and sort them.

The database _might_ be able to use an index and avoid fetching the entire row.  Our example query uses the primary key index but it still crawls.

## Faster Chunking With ChunkById

It turns out it's easy to page through results without using offset. All we have to do is keep track of the last ID we saw, then filter the results so we only fetch rows we haven't seen.  If the results are ordered by ID in ascending order we can use a simple `WHERE ID > :last_id` clause to filter.  The query ends up looking like this:

```sql
select * from `users` where `id` > :last_id order by `id` asc limit 500;
```

With this technique the last page loads as quickly as the first.

Laravel makes this effortless.  Instead of calling `chunk` call the `chunkById` method ([added in 5.2](https://github.com/laravel/framework/pull/12861)).  The query builder will add the where clause, the order by, and the limit.

The API is exactly the same as `chunk`.  As long as you aren't using any custom order by clauses it's a 4 character change.

[^1]: YMMV.  I tested a table with 1,632,576 rows using the null scout driver.  The elapsed time went from 29 minutes and 57 seconds to only 28 seconds.