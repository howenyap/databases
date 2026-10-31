# CMU 15-445 #14: Query Execution Part 2: answers

Questions only: [quiz.md](quiz.md). Interactive version: [quiz.html](quiz.html).

## 1. Which process model does Postgres use?

_Around 10:19_

- A) Process per worker
- B) Thread per worker
- C) Embedded
- D) Coroutine per worker

**Answer:** A. Postgres forks one OS process per worker, and the workers share data through shared memory. It predates standardized POSIX threads, when fork was the only thing that behaved the same on every Unix.

## 2. What's an advantage of process-per-worker over thread-per-worker?

_Around 23:33_

- A) Cheaper context switches
- B) No need for shared memory
- C) One worker crashing doesn't take down the whole DBMS
- D) The DBMS controls scheduling instead of the OS

**Answer:** C. Each process has its own address space, so a segfault kills only that worker, and the postmaster replaces it. In a threaded DBMS, one crashing thread takes down the whole process. The other options describe advantages of threads.

## 3. In an embedded DBMS like SQLite or DuckDB, who provides the threads that run queries?

_Around 19:24_

- A) A dedicated DBMS worker pool
- B) The OS, through a dispatcher process
- C) The application that links the library
- D) A separate database server

**Answer:** C. The DBMS is a library in the application's address space, and the application's thread calls in and runs the query. SQLite has no worker threads of its own, not even background ones.

## 4. Which kind of parallelism lowers the latency of one long-running analytical query?

_Around 29:50_

- A) Inter-query
- B) Intra-query
- C) Both equally
- D) Neither

**Answer:** B. Intra-query splits one query across workers, which cuts its latency. Inter-query runs different queries at once, which raises throughput. MySQL has only inter-query, so one query uses one core even on a 1000-core machine.

## 5. Which type of exchange operator combines results from several workers into one output stream?

_Around 39:53_

- A) Distribute
- B) Repartition
- C) Gather
- D) Broadcast

**Answer:** C. Gather takes N streams to 1. Distribute splits 1 stream into N, and Repartition reshuffles N streams into M.

## 6. A join's output is pipelined straight into a projection on another worker, without materializing in between. What's this called?

_Around 42:30_

- A) Intra-operator (horizontal)
- B) Inter-operator (vertical)
- C) Bushy
- D) Inter-query

**Answer:** B. Inter-operator (vertical, or pipelined) parallelism runs different operators at the same time on different workers, with tuples streaming between them. The lecture's example puts an expensive UDF on a second worker.

## 7. Select all that are true:

_Around 45:39. Select all that apply._

- A) Intra-operator runs the same operator on disjoint subsets of the data
- B) Bushy is a hybrid of intra-operator and inter-operator
- C) The three types are mutually exclusive; a DBMS must pick one
- D) Vertical parallelism is widely used in stream processing

**Answer:** A, B, D. A, B and D are true. The types aren't mutually exclusive: real systems combine them in one plan, which is what bushy parallelism is.

## 8. A 1 TB database is mirrored with RAID 1 across 4 disks. Which is true?

_Around 52:31_

- A) Each disk needs about 250 GB
- B) Reads and writes both get about 4x faster
- C) Each disk needs the full 1 TB; reads can scale but writes don't
- D) It gives no fault tolerance

**Answer:** C. In a mirror, every disk holds a full copy. Any disk can serve a read, but every write must reach all four disks and fsync on each before it's acknowledged.

## 9. With multi-disk parallelism (RAID or a storage appliance), why can't the DBMS assign workers to specific disks?

_Around 50:27_

**Answer:** The striping or mirroring happens in the OS or hardware below the DBMS, and it's transparent. The DBMS sees one logical device and doesn't know which disk holds which page.

## 10. Why might adding more worker threads not speed up queries at all, and what does the lecture suggest doing about it?

_Around 48:18_

**Answer:** If disk I/O is the bottleneck, extra threads just wait on the same disk. Use I/O parallelism: spread the database across several devices, with multi-disk setups (RAID) or database partitioning.
