# psqlrc-helpers
A pack of `psql` helper commands to maintain a PostgreSQL.
- `psql` autocomplete support.
- No server install needed. Everything is in your local `~/.psqlrc`
- All commands are intentionally read-only, won't screw anything up.

It is implemented via `\set` metacommand of the [psql](https://www.postgresql.org/docs/current/app-psql.html#AEN88713).

# Table of contents

  - [Installation](#installation)
  - [Commands](#commands)
    - [`:maintenance`](#maintenance)
    - [`:help`](#help)
    - [`:fullexplain`, or `:fe`](#fullexplain-or-fe)
    - [`:scanstat`, or `:ss`](#scanstat-or-ss)
    - [`:queries`, or `:q`](#queries-or-q)
    - [`:listens`, or `:l`](#listens-or-l)
    - [`:extensions`, or `:e`](#extensions-or-e)

## Installation

Get `psqlrc` file from this repo and add its content to your `~/.psqlrc`

```
git clone https://github.com/hom3chuk/psqlrc-helpers.git
cd psqlrc-helpers
cat psqlrc >> ~/.psqlrc
```

That's it! All new `psql` sessions will have `psqlrc-helpers` commands available.

Or you can just copy the SQL for the command you need from this readme.

## Commands

### `:maintenance`

This command shows a brief report on the current database status:
- number of tables with more than 50% of non-index scans (`:ss`)
- number of outdated server extensions (`:e`)
- number of long-running (that is, > 1 minute) queries (`:q`)

This is the command you want to occasionally run when checking what is up with the server.

#### SQL

```postgresql
with slow_queries as (select count(*)
                      from pg_stat_activity
                      where state <> 'idle'
                        and query not ilike '<idle>'
                        and query not ilike '%pepaka_query_filter%'
                        and query not ilike 'listen %'
                        and backend_type <> 'walsender'
                        and query_start < now() - interval '1 minute'),
     stale_extensions as (select count(*) as count
                          from pg_available_extensions
                          where installed_version <> default_version),
     seq_scans as (select count(*) as count
                   from pg_stat_user_tables
                   where coalesce(seq_scan, 0) > coalesce(idx_scan, 0))
select case
         when stale_extensions.count > 0 then 'BAD: You have ' || stale_extensions.count ||
                                              ' outdated extensions. Call `:e` for more.'
         else 'GOOD: All extensions are up to date' end as report
from stale_extensions
union
select case
         when slow_queries.count > 0 then 'BAD: You have ' || slow_queries.count ||
                                          ' long running queries (> 60s). Call `:q` for more.'
         else 'GOOD: No long running queries' end
from slow_queries
union
select case
         when seq_scans.count > 0 then 'BAD: You have ' || seq_scans.count ||
                                       ' tables with < 50% index scans. Call `:ss` for more.'
         else 'GOOD: No <50% index scan tables' end
from seq_scans;
```

### `:help`

Lists all available commands

#### SQL
```postgresql
select *
from (values (':fullexplain', ':fe', 'runs EXPLAIN (ANALYZE, BUFFERS, VERBOSE)', ':fullexplain select * from pg_stat_activity;'),
             (':scanstat', ':ss', 'lists index/seq scan stats per table', ':ss'),
             (':queries', ':q', 'lists currently running queries ordered by execution time desc', ':q'),
             (':listens', ':l', 'lists all active LISTEN channels', ':l'),
             (':extensions', ':e', 'lists all installed extensions, outdated first', ':e'),
             (':help', '', 'prints this message', ':help')) as a(command, alias, description, example);
```

### `:fullexplain`, or `:fe`

Runs a `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` for a given query.

This is the only command that requires extra input, the query to run `EXPLAIN` against:

```
:fe select * from pg_stat_activity
```

#### SQL

```postgresql
explain (analyze, buffers, verbose) 
```

### `:scanstat`, or `:ss`

Lists [indexed/seq scan](https://www.depesz.com/2013/04/27/explaining-the-unexplainable-part-2/) ratio per table.

#### SQL

```postgresql
select relname,
       idx_scan,
       seq_scan,
       n_live_tup,
       trunc(coalesce(100.0 * idx_scan / (case when (idx_scan + seq_scan) = 0 then 1 else (idx_scan + seq_scan) end),
                      0.00), 2) as idx_scan_percent
from pg_stat_user_tables
order by 5, 3 desc, 1;
```

### `:queries`, or `:q`

Lists currently running queries ordered by execution time desc.

#### SQL

```postgresql
select pid, now() - query_start as "time", usename, query
from pg_stat_activity
where state <> 'idle'
  and query not ilike '<idle>'
  and query not ilike '%pepaka_query_filter%'
  and query not ilike 'listen %'
  and backend_type <> 'walsender'
  and query_start < now() - interval '1 minute'
order by 2 desc nulls last, 1 desc;
```

### `:listens`, or `:l`

Lists all active LISTEN channels

#### SQL
```postgresql
select pid, now() - query_start as "time", usename, query
from pg_stat_activity
where query ilike 'listen %'
order by 2 desc nulls last, 1 desc;
```

### `:extensions`, or `:e`

Lists all extensions installed on the server, outdated first.

#### SQL

```postgresql
select name,
       installed_version,
       default_version as available_version,
       case
         when (installed_version <> default_version)
           then 'outdated'
         else 'latest' end as status
from pg_available_extensions
where installed_version is not null
order by 4 desc, 1;
```
