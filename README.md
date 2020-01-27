# Index Skip Scan

The goal of this project is to add support for "Index Skip Scan" in PostgreSQL 13.

Index Skip Scan will allow PostgreSQL to optimize `DISTINCT` queries from

```sql
CREATE TABLE t1 (a integer PRIMARY KEY, b integer);
CREATE INDEX idx_t1_b ON t1 (b);
INSERT INTO t1 (SELECT i, i % 3 FROM generate_series(1, 10000000) as i);
ANALYZE;
EXPLAIN (ANALYZE, VERBOSE, BUFFERS ON) SELECT DISTINCT b FROM t1;
 HashAggregate  (cost=169247.71..169247.74 rows=3 width=4) (actual time=4104.099..4104.099 rows=3 loops=1)
   Output: b
   Group Key: t1.b
   Buffers: shared hit=44248
   ->  Seq Scan on public.t1  (cost=0.00..144247.77 rows=9999977 width=4) (actual time=0.059..1050.376 rows=10000000 loops=1)
         Output: a, b
         Buffers: shared hit=44248
 Planning Time: 0.157 ms
 Execution Time: 4104.155 ms
(9 rows)
```

to

```sql
CREATE TABLE t1 (a integer PRIMARY KEY, b integer);
CREATE INDEX idx_t1_b ON t1 (b);
INSERT INTO t1 (SELECT i, i % 3 FROM generate_series(1, 10000000) as i);
ANALYZE;
EXPLAIN (ANALYZE, VERBOSE, BUFFERS ON) SELECT DISTINCT b FROM t1;
 Index Only Scan using idx_t1_b on public.t1  (cost=0.43..1.30 rows=3 width=4) (actual time=0.027..0.060 rows=3 loops=1)
   Output: b
   Scan mode: Skip scan
   Heap Fetches: 3
   Buffers: shared hit=12 read=3
 Planning Time: 0.204 ms
 Execution Time: 0.070 ms
(7 rows)
```

**Current commit message:**

```
Index skip scan

Implementation of Index Skip Scan (see Loose Index Scan in the wiki [1])
on top of IndexOnlyScan and IndexScan. To make it suitable for both
situations when there are small number of distinct values and
significant amount of distinct values the following approach is taken -
instead of searching from the root for every value we're searching for
then first on the current page, and then if not found continue searching
from the root.

Original patch and design were proposed by Thomas Munro [2], revived and
improved by Dmitry Dolgov and Jesper Pedersen.

[1] https://wiki.postgresql.org/wiki/Loose_indexscan
[2] https://www.postgresql.org/message-id/flat/CADLWmXXbTSBxP-MzJuPAYSsL_2f0iPm5VWPbCvDbVvfX93FKkw%40mail.gmail.com

Author: Jesper Pedersen, Dmitry Dolgov
Reviewed-by: Thomas Munro, David Rowley, Floris Van Nee, Kyotaro Horiguchi, Tomas Vondra, Peter Geoghegan
```

**Status:**

Beta (v33)

## Discussion

The discussion for this patch is located on [pgsql-hackers](https://www.postgresql.org/list/pgsql-hackers/).

**Thread:**

* [Index Skip Scan](https://www.postgresql.org/message-id/flat/707b6f68-16fa-7aa7-96e5-eeb4865e6a30%40redhat.com)

**Background:**

* [Loose Index Scan](https://wiki.postgresql.org/wiki/Loose_indexscan)
* [Previous discussion](https://www.postgresql.org/message-id/flat/CADLWmXXbTSBxP-MzJuPAYSsL_2f0iPm5VWPbCvDbVvfX93FKkw%40mail.gmail.com#CADLWmXXbTSBxP-MzJuPAYSsL_2f0iPm5VWPbCvDbVvfX93FKkw@mail.gmail.com)
