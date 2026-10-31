# #13: Query Execution I

## Resources

- [Video](https://www.youtube.com/watch?v=E-UUd6cB57w)
- [Slides](https://15445.courses.cs.cmu.edu/fall2025/slides/13-queryexecution1.pdf)
- [Notes](https://15445.courses.cs.cmu.edu/fall2025/notes/13-queryexecution1.pdf)
- Readings: Chapter 15.1-15.3, 15.7
- Quiz: [interactive](quiz/quiz.html) (open it in a browser) · [questions](quiz/quiz.md) · [answers](quiz/answer.md)

## Papers

- Graefe, *Volcano: An Extensible and Parallel Query Evaluation System*, IEEE TKDE 1994. The iterator model is named after it. [doi](https://doi.org/10.1109/69.273032)
- Boncz, Zukowski & Nes, *MonetDB/X100: Hyper-Pipelining Query Execution*, CIDR 2005. The origin of the vectorized model. [PDF](https://www.cidrdb.org/cidr2005/papers/P19.pdf)
- Moerkotte, *Small Materialized Aggregates: A Light Weight Index Structure for Data Warehousing*, VLDB 1998. The zone maps paper. [PDF](https://www.vldb.org/conf/1998/p476.pdf)
- *A Well-Intentioned Query and the Halloween Problem*, IEEE Annals of the History of Computing 2002. The history of the Halloween problem; IBM never published the 1976 fix. [doi](https://doi.org/10.1109/mahc.2002.1010071)

## Summary

The timestamps link to the part of the video they cover.

### 0. Pipelines and pipeline breakers ([5:40](https://www.youtube.com/watch?v=E-UUd6cB57w&t=340s))

- A query plan is a tree (strictly, a DAG) of operators, and data flows up it.
- A **pipeline** is a run of operators that tuples can flow through without stopping.
- A **pipeline breaker** needs all of its input before it can output anything. Examples are the build side of a hash join and sort.

### 1. Processing models: iterator, materialization, vectorized ([8:50](https://www.youtube.com/watch?v=E-UUd6cB57w&t=530s))

- Each model has two parts. **Control flow** decides how an operator gets told to run. **Data flow** decides how its output moves to the next operator.
- **Iterator / Volcano** ([11:53](https://www.youtube.com/watch?v=E-UUd6cB57w&t=713s))
  - Every operator has `open()`, `next()` and `close()`. Each `next()` returns one tuple, or EOF when there are no more.
  - It's simple, easy to debug, and makes `LIMIT` easy (you just stop calling `next()`). BusTub uses it, so you build it in Project 3.
  - The downside is one function call per tuple, which gets expensive over a billion tuples.
  - Used by Postgres, MySQL and SQLite.
- **Materialization** ([21:20](https://www.youtube.com/watch?v=E-UUd6cB57w&t=1280s))
  - One call returns all of an operator's output at once.
  - It's bad when you need only part of the result, like a `LIMIT 10` over a billion rows. The fix is **operator fusion**: push filters and projections down into the scan.
  - It fits in-memory OLTP, where queries touch few tuples. MonetDB introduced it. It's rare today.
- **Vectorized / batch** ([27:20](https://www.youtube.com/watch?v=E-UUd6cB57w&t=1640s))
  - `next()` returns a batch of tuples, usually around 1024 (DuckDB's size).
  - You get fewer calls, and tight loops without branches that can use SIMD or a GPU.
  - Every modern OLAP system uses it. It came from a CWI researcher who later co-founded Snowflake.
- No one mixes processing models inside one engine because it's too much to maintain. Oracle and SQL Server get around this by shipping a separate column-store engine.

### 2. Plan direction: pull vs push ([32:15](https://www.youtube.com/watch?v=E-UUd6cB57w&t=1935s))

- **Pull (top-down):** the root calls `next()`, and data gets pulled up the tree. It's the most common approach, it's simple, and `LIMIT` is easy.
- **Push (bottom-up):** a **scheduler** runs pipelines as tasks in dependency order and tells each task where its input lives. This separates control flow from data flow and gives the scheduler a view of the whole plan.
- Push-based systems include HyPer, Umbra, DuckDB (which switched to push), ClickHouse, Snowflake and Firebolt.
- Plan direction is **independent** of the processing model, so any combination works.

### 3. Access methods: the scans at the bottom of the plan ([42:04](https://www.youtube.com/watch?v=E-UUd6cB57w&t=2524s))

- **Sequential scan** is the fallback, and it keeps a cursor so it can resume.
  - Earlier lectures already covered ways to speed it up: compression, prefetching, sorting and clustering, late materialization, materialized views, result caching, and code specialization.
- **Data skipping**
  - **Approximate queries** use sampling and return lossy answers. You have to ask for them explicitly, because the database can't know whether an approximate answer is acceptable for your use.
  - **Zone maps** (also called SMAs, small materialized aggregates) store min, max, null count and similar stats for each block. With `val > 600` and a block whose max is 400, the scan skips that block. They're always correct, so you always use them.
- **Index scan:** choosing which index to use depends on how selective each predicate is (the example compares an age index with a department index). That choice is the optimizer's job, covered next week. A hash index can't answer range predicates like `<`.
- **Multi-index scan:** look up several indexes and combine the record IDs, using **intersection** for `AND` and **union** for `OR`. Postgres calls this a *bitmap scan* and MySQL calls it an *index merge*.

### 4. Modification queries ([56:55](https://www.youtube.com/watch?v=E-UUd6cB57w&t=3415s))

- Inserts, updates and deletes reuse the existing scan operators to find the tuples they change.
- Inserts can build the tuple inside the operator or receive it from below in the plan. The second way is why `INSERT ... SELECT` and `CREATE TABLE AS` compose easily.
- **Halloween problem** ([59:36](https://www.youtube.com/watch?v=E-UUd6cB57w&t=3576s))
  - An update moves a tuple's physical location, for example further along an index on salary. The scan then finds the same logical tuple again and updates it twice.
  - IBM found it in System R in 1976. It was a Friday that happened to be Halloween, so they went out drinking and fixed it the next week.
  - The fix is to track which tuples you've already modified. You need this in Project 3.

### 5. Expression evaluation ([63:29](https://www.youtube.com/watch?v=E-UUd6cB57w&t=3809s))

- `WHERE` and join clauses are stored as **expression trees**. Evaluating one means walking the tree for every tuple, with an execution context that holds the current tuple, the query parameters and the schema.
- Walking the tree is slow because of all the indirection. It's better to evaluate the expression directly, using SIMD over batches, or to **JIT-compile** it.
- **Demo** ([69:55](https://www.youtube.com/watch?v=E-UUd6cB57w&t=4195s))
  - Postgres JIT-compiles only the `WHERE` clause with LLVM. That took the query from about 4s to about 2.7s, with about 381ms spent compiling.
  - DuckDB ran the same query in under 1s on a single thread, with no JIT. It's a vectorized, push-based column store.
  - The lesson: a design built for OLAP beats adding a JIT to a row store.
- **Constant folding:** compute `UPPER('wu-tang')` once and use the result as a constant.
- **Common subexpression elimination:** when the same subtree appears twice, such as the same `STRPOS(...)` on both sides of an `OR`, compute it once and reuse the result.
- HyPer and Umbra start by emitting assembly that runs right away. They compile with LLVM in the background and swap the compiled code in if it finishes before the query does.

### Takeaways ([78:39](https://www.youtube.com/watch?v=E-UUd6cB57w&t=4719s))

- For OLAP, use the vectorized model. Push vs pull is an engineering tradeoff.
- For OLTP, use index scans as much as you can. If the data fits in memory, the materialization model works well.
- Next lecture covers running all of this in parallel.
