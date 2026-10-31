# CMU 15-445 #14: Query Execution Part 2: questions

Answers: [answer.md](answer.md). Interactive version: [quiz.html](quiz.html).

## 1. Which process model does Postgres use?

_Around 10:19_

- A) Process per worker
- B) Thread per worker
- C) Embedded
- D) Coroutine per worker

## 2. What's an advantage of process-per-worker over thread-per-worker?

_Around 23:33_

- A) Cheaper context switches
- B) No need for shared memory
- C) One worker crashing doesn't take down the whole DBMS
- D) The DBMS controls scheduling instead of the OS

## 3. In an embedded DBMS like SQLite or DuckDB, who provides the threads that run queries?

_Around 19:24_

- A) A dedicated DBMS worker pool
- B) The OS, through a dispatcher process
- C) The application that links the library
- D) A separate database server

## 4. Which kind of parallelism lowers the latency of one long-running analytical query?

_Around 29:50_

- A) Inter-query
- B) Intra-query
- C) Both equally
- D) Neither

## 5. Which type of exchange operator combines results from several workers into one output stream?

_Around 39:53_

- A) Distribute
- B) Repartition
- C) Gather
- D) Broadcast

## 6. A join's output is pipelined straight into a projection on another worker, without materializing in between. What's this called?

_Around 42:30_

- A) Intra-operator (horizontal)
- B) Inter-operator (vertical)
- C) Bushy
- D) Inter-query

## 7. Select all that are true:

_Around 45:39. Select all that apply._

- A) Intra-operator runs the same operator on disjoint subsets of the data
- B) Bushy is a hybrid of intra-operator and inter-operator
- C) The three types are mutually exclusive; a DBMS must pick one
- D) Vertical parallelism is widely used in stream processing

## 8. A 1 TB database is mirrored with RAID 1 across 4 disks. Which is true?

_Around 52:31_

- A) Each disk needs about 250 GB
- B) Reads and writes both get about 4x faster
- C) Each disk needs the full 1 TB; reads can scale but writes don't
- D) It gives no fault tolerance

## 9. With multi-disk parallelism (RAID or a storage appliance), why can't the DBMS assign workers to specific disks?

_Around 50:27_

## 10. Why might adding more worker threads not speed up queries at all, and what does the lecture suggest doing about it?

_Around 48:18_
