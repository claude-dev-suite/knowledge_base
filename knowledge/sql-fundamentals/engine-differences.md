# SQL Engine Differences That Change What a Query Means

> No upstream. Each vendor documents its own behaviour; the comparison is not
> published anywhere, because no vendor benefits from stating where their
> defaults differ from everyone else's.

## Why this page exists

SQL looks portable and is not, at exactly the level that decides whether a
query is correct. The same statement, moved between engines, can commit
different rows, take a different lock, or match a different set of strings —
without any error.

These are the differences that change **meaning or safety**, not syntax
conveniences.

## Default isolation level

This is the single largest source of "correct in test, wrong in production".

| Engine | Default | Prevents |
|---|---|---|
| PostgreSQL | READ COMMITTED | dirty reads |
| Oracle | READ COMMITTED | dirty reads |
| SQL Server | READ COMMITTED | dirty reads (locking-based unless RCSI is on) |
| MySQL / InnoDB | **REPEATABLE READ** | dirty and non-repeatable reads |
| SQLite | SERIALIZABLE (single writer) | everything, by having one writer |

**What follows for review.** A check-then-act — read a count, decide, then
insert — is safe under SERIALIZABLE and **nowhere else**. Under both common
defaults another transaction can insert between the read and the write. MySQL's
REPEATABLE READ stops the *re-read* from changing, which is not the same as
stopping the other transaction from committing.

Two consequences people get wrong:

- Raising the level is not free: SERIALIZABLE means the application must handle
  serialisation failures and retry. Code that does not retry has traded a
  silent wrong answer for a visible error, which is better but is not "fixed".
- A unique constraint is usually cheaper and stronger than any isolation level,
  because it holds regardless of what the application forgot.

**SQL Server specifically**: its default uses shared locks rather than row
versioning unless `READ_COMMITTED_SNAPSHOT` is on, so readers can block writers.
The same code has different concurrency behaviour on two SQL Server databases.

## NULL, and where engines disagree

Three-valued logic is standard and behaves the same everywhere: `x NOT IN
(1, NULL)` is UNKNOWN, so the row is filtered out and the query returns
nothing. That trap is universal — see `sql-fundamentals/basics`.

What differs is **uniqueness**:

| Engine | Multiple NULLs in a UNIQUE column |
|---|---|
| PostgreSQL | allowed (unless `NULLS NOT DISTINCT`, 15+) |
| MySQL | allowed |
| Oracle | allowed for single-column; composite differs |
| SQL Server | **at most one** |

A "one active row per user" constraint expressed as a nullable column plus
UNIQUE therefore enforces something different on SQL Server than on the others.

Oracle has a second, larger divergence: it treats the **empty string as NULL**.
`WHERE name = ''` never matches, and a `NOT NULL` column happily rejects `''`.
Code ported to or from Oracle changes meaning silently.

## Collation and case sensitivity

| Engine | Default comparison |
|---|---|
| MySQL | **case-insensitive** (`utf8mb4_0900_ai_ci` and predecessors) |
| PostgreSQL | case-sensitive |
| SQL Server | depends on the instance/database collation, commonly CI |
| Oracle | case-sensitive by default |

**What follows for review.** A uniqueness assumption — "no two users with the
same email" — holds differently: on MySQL `Bob@x.com` and `bob@x.com` collide,
on PostgreSQL they do not. A login lookup that "works" on MySQL may become
case-sensitive on migration, locking users out. Normalising in the application,
or a functional index on `lower(...)`, makes the behaviour explicit rather than
inherited.

Sorting inherits the same problem: `ORDER BY name` produces a different order
under different collations, which matters for pagination stability.

## DDL and locking during migrations

| Operation | PostgreSQL | MySQL / InnoDB |
|---|---|---|
| `ADD COLUMN ... NOT NULL DEFAULT` | rewrites the table before 11; metadata-only from 11 | instant from 8.0.12 with `ALGORITHM=INSTANT`, otherwise a rebuild |
| `CREATE INDEX` | blocks writes unless `CONCURRENTLY` | `ALGORITHM=INPLACE` allows concurrent DML in most cases |
| `CONCURRENTLY` inside a transaction | **not allowed** | n/a |

The last row is the one that bites: migration tools wrap each migration in a
transaction by default, so `CREATE INDEX CONCURRENTLY` fails unless the tool is
told not to. The workaround is per-tool, and forgetting it turns a
zero-downtime migration into a table lock.

## Identifier quoting and case folding

- PostgreSQL folds unquoted identifiers to **lower** case.
- Oracle folds them to **upper** case.
- MySQL's table-name case sensitivity depends on `lower_case_table_names`,
  which in practice depends on the host filesystem.

A quoted identifier is preserved verbatim everywhere — which is why a schema
created with quoted mixed-case names is painful to query from code that does
not quote.

## What to establish before commenting

1. **Which engine and version.** Almost every row above has a version boundary.
2. **The effective isolation level**, including whether the application or the
   connection pool overrides the default.
3. **The collation**, if the query relies on matching or uniqueness of text.
4. **Whether the SQL is hand-written or ORM-generated.** If generated, the fix
   belongs in the mapping, not in the statement.

State the engine once in the review. Half of these findings do not exist on the
other engines, and a comment that does not say which one it assumes is not
actionable.
