# #14: Query Execution II

## Resources

- [Video](https://www.youtube.com/watch?v=Kzf1hGjtZOU)
- [Slides](https://15445.courses.cs.cmu.edu/fall2025/slides/14-queryexecution2.pdf)
- [Notes](https://15445.courses.cs.cmu.edu/fall2025/notes/14-queryexecution2.pdf)
- Readings: Chapter 22
- Quiz: [interactive](quiz/quiz.html) (open it in a browser) · [questions](quiz/quiz.md) · [answers](quiz/answer.md)
- Guest speaker: [speaker.md](speaker.md)

## Papers

- Graefe, *Encapsulation of Parallelism in the Volcano Query Processing System*, SIGMOD 1990. The origin of the exchange operator. [PDF on ACM](https://dl.acm.org/doi/pdf/10.1145/93605.98720)
- Stonebraker & Pavlo, *What Goes Around Comes Around... And Around...*, SIGMOD Record 2024. Mentioned in the guest talk. [PDF](https://db.cs.cmu.edu/papers/2024/whatgoesaround-sigmodrec2024.pdf) · [doi](https://doi.org/10.1145/3685980.3685984)
- Nguyen & Leis, *Why Files If You Have a DBMS?*, ICDE 2024. Mentioned in the guest talk. [PDF](https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/blob.pdf) · [doi](https://doi.org/10.1109/icde60146.2024.00297)

## Summary

The timestamps link to the part of the video they cover. On YouTube the video is titled "Parallel Query Engine Architectures", and the last 15 minutes are a guest talk from SpiralDB.

### 0. Why run queries in parallel ([3:03](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=183s))

- The goal is to use more cores, disks and memory than one worker can, for better throughput and latency. Spreading data across devices can also give fault tolerance.
- SQL is declarative, so the same query runs unchanged whether the DBMS uses one core or a thousand. The plan, not the query, decides where data moves.
- **Parallel vs distributed** ([6:09](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=369s))
  - A **parallel** DBMS has its resources physically close, in one machine or rack. Communication is fast, cheap and reliable.
  - A **distributed** DBMS has resources far apart: other racks, regions, or (per Nvidia's recent announcement) space. Messages are slow and can get lost. That's the end of the semester.
  - This lecture assumes a single node where every read and write arrives.

### 1. Process models ([8:43](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=523s))

- A **worker** is the unit that runs tasks for clients. It can also be a background task, such as LSM compaction. The lecture says "worker", not "thread", because the unit depends on the process model.
- **Process per worker** ([11:21](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=681s))
  - Each worker is a separate OS process created with `fork`, with its own address space. The OS schedules them.
  - The application talks to a **dispatcher** (Postgres calls it the *postmaster*), which hands it a worker to connect to.
  - Workers share data through shared memory or, at worst, IPC such as pipes and signals.
  - **Why old systems did this** ([13:26](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=806s)): POSIX threads weren't standardized until the mid-90s. Each Unix (BSD, HP-UX, AIX, Solaris) had its own thread library with different semantics, but `fork` behaved the same everywhere. Postgres and its forks still work this way because rewriting would be too hard.
- **Thread per worker** ([15:31](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=931s))
  - One process with many threads, usually from a thread pool rather than a `pthread_create` per request. Threads share one address space, so they communicate by reading each other's memory, but shared structures still need latches.
  - It's cheaper than processes, and the DBMS can control scheduling itself, for example with coroutines.
  - Oracle started with processes and switched to threads by default in 2014. Every new system uses threads.
- **Embedded** ([18:53](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1133s))
  - The DBMS is a library inside the application's address space and has no worker pool. The application's own thread calls in, runs the query, and returns up the stack.
  - Examples: SQLite (no background workers at all), Berkeley DB (the first, bought by Oracle in 2006), RocksDB, LevelDB. DuckDB can optionally start its own threads.
- **Scheduling** ([21:28](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1288s)): the DBMS decides how many workers a query gets, which ones, and when. It knows the query and the resources, so it should never leave this to the OS. Oracle, SQL Server and DB2 do it well. Postgres and MySQL let the OS schedule.
- **Tradeoff** ([23:02](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1382s)): threads have lower overhead and no shared-memory management. The cost is that a segfault in one thread kills the whole DBMS. In Postgres, a crashed worker process just gets replaced by the postmaster.

### 2. Inter-query parallelism ([26:41](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1601s))

- Run different queries at the same time on different workers. This raises **throughput**.
- Most systems schedule first-come, first-served.
- Read-only queries need almost no coordination. Concurrent updates do, which is concurrency control from lecture 17 onward.
- **Inter-query doesn't imply intra-query** ([28:47](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1727s)): MySQL runs many queries at once, but each query uses one thread, so on a 1000-core machine one query still uses one core. Presto does both.

### 3. Intra-query parallelism ([29:50](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=1790s))

- Run parts of one query at the same time. This lowers its **latency**.
- Almost every operator has a parallel version through divide and conquer. For example, in a parallel Grace hash join, each worker takes its own partition level, and workers don't need to talk to each other.
- There are three types, and they aren't mutually exclusive.

#### Intra-operator (horizontal) ([33:32](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2012s))

- Split the data into disjoint subsets (by page, by range, or by hashing a key), and run an instance of the same operator on each subset.
- This is by far the most common type. When a vendor says "parallel database", they almost always mean this.
- An **exchange operator** ([34:03](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2043s)) sits between parallel parts of the plan and coordinates them. It has no equivalent in relational algebra. Think of it as a barrier: nothing above it runs until every worker below finishes. It doesn't always have to block, and it doesn't when the operator above can start on partial results.
- **Example** ([36:41](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2201s)): `A ⋈ B` with a filter on each table.
  - Three workers each filter a third of A and insert into one shared hash table. An exchange waits until all three finish.
  - Then three workers each filter a third of B and probe the table. Probing before the build finishes would miss matches.
  - A final exchange gathers the results.
- **Exchange types** ([39:53](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2393s)), named with SQL Server's terms:
  - **Gather:** N input streams into 1 output stream.
  - **Distribute:** 1 input stream split into N output streams.
  - **Repartition** ([40:57](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2457s)): N streams into M streams, where M doesn't have to equal N. For example, use many workers for a scan, then fewer after filters shrink the data, because moving data between workers isn't free. BigQuery/Dremel repartitions at every stage to resize the worker count on the fly.

#### Inter-operator (vertical, pipelined) ([42:30](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2550s))

- Different operators of the same plan run at the same time on different workers. Tuples stream from one to the next without being materialized in between.
- **Example** ([44:04](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2644s)): one worker runs a join and emits matches. A second worker reads that stream and calls an expensive UDF on each tuple. The second worker blocks while it waits for input.
- It isn't free: tuples get copied between workers, which costs more than running both steps in one loop.
- It's less common in regular DBMSs, because pipeline breakers would leave upper workers idle. It's widely used in **stream processing**, where data never stops arriving and queries run forever.

#### Bushy ([45:39](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2739s))

- A hybrid of intra-operator and inter-operator: different workers run different segments of the plan at the same time, and each segment can also be split across workers.
- Example: `(A ⋈ B)` and `(C ⋈ D)` run in parallel, then join their results.
- This plan shape is called a *bushy join*. Most optimizers only produce *left-deep* plans, because searching every bushy plan is too expensive. That's next lecture.
- Bushy parallelism mostly shows up in expensive systems such as SQL Server, DB2 and Oracle.

### 4. I/O parallelism ([48:18](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2898s))

- More workers don't help if the **disk** is the bottleneck: they all wait on the same device. Storage like S3 is always much slower than the CPU.
- The fix is to spread the database across several storage devices so that reads and writes hit different disks in parallel.

#### Multi-disk parallelism ([49:21](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=2961s))

- You can put one database per disk, one table per disk, or split one table across disks.
  - Oracle lets you choose where each table goes, and you can put the log on one disk and the tables on another.
  - Postgres offers *tablespaces*.
- **Usually it's transparent** ([50:27](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3027s)): when the OS or hardware stripes and mirrors data, the DBMS sees one logical device. It doesn't know which disk holds which page, so it can't assign workers to disks.
- **RAID 0, striping** ([51:29](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3089s))
  - Pages go round-robin across the disks, and each page lives on exactly one disk.
  - Reads and writes on different disks run in parallel.
  - Losing one of three disks loses a third of the database.
- **RAID 1, mirroring** ([52:31](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3151s))
  - Every page is written to every disk, and a write isn't acknowledged until every disk confirms its `fsync`.
  - Reads are fast because any disk can serve any page. Writes are slower because every disk has to be updated.
- **Hardware or software** ([53:33](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3213s)): a storage controller can do RAID invisibly. The better approach, used by high-end systems like Oracle, is to manage layout inside the DBMS, so it knows exactly where data lives and what losing a disk costs.
- **Tradeoffs** ([54:36](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3276s)): performance, durability and capacity.
  - Mirroring a 1 TB database needs 1 TB on every disk.
  - Places that care about their data, like Visa and MasterCard, just pay for more disks, so capacity matters less to them than speed and durability.
  - Most systems today use S3, which has unlimited capacity and handles durability for you.

#### Database partitioning ([56:12](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3372s))

- Split a table or database into disjoint subsets, by range or by hash, and put them on different devices.
- The DBMS controls it fully, so it can choose placement better than transparent RAID can. The application's SQL doesn't change.
- A column store is already a form of partitioning: each column is stored in its own files.
- Covered properly with distributed databases.

### Takeaways ([58:18](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3498s))

- Parallel execution is everywhere, because modern hardware gets faster by adding cores, not clock speed.
- More workers also hide latency: while one waits on disk or the network, others keep running.
- The DBMS knows what every query needs, so it should schedule workers itself instead of letting the OS.
- Next lecture covers query optimization. Pavlo's "dirty secret" ([60:54](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3654s)) is that every query optimizer is bad at estimating, because it has to choose a plan before it sees the data.

### Guest talk: SpiralDB and Vortex ([61:25](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3685s))

About the speaker and company, with links: [speaker.md](speaker.md).

- **Hardware history:** clock speeds stopped rising around 2005, so chips added cores instead. Hadoop, Mesos and Kubernetes were about managing big pools of identical cores and machines.
- **What changed by 2025** ([63:59](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3839s))
  - Hardware is heterogeneous now, with GPUs everywhere.
  - Network cards now have more bandwidth than PCIe. The old speed order (memory faster than disk, disk faster than the network) still holds for latency, but not for bandwidth.
  - Storage is moving to object stores and NVMe.
- **Three eras of databases** ([65:30](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=3930s))
  1. Human-scale input and output, like Postgres updating one row of a profile.
  2. Machine-scale input with human-scale output, like MapReduce, data lakes and lakehouses.
  3. Machine-scale *output*, like streaming terabytes of results into GPUs for training.
- **Spiral** ([68:15](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=4095s)) is a columnar store built for object storage, with primary keys and indexes. Its goal is to load more bytes per second into GPUs straight from S3 than from Parquet on a local disk, which means saturating a network card.
- **Vortex** ([71:24](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=4284s)) is Spiral's open file format, now a Linux Foundation project.
  - It's pitched as a faster replacement for Parquet: over 100x faster random access, and about 5-18x faster writes and scans.
  - Its scan operator compiles to run on the CPU or directly on the GPU.
- **CPU to GPU** ([75:04](https://www.youtube.com/watch?v=Kzf1hGjtZOU&t=4504s)): the main change is that GPU kernels need to know each operator's output size in advance, which shifts where pipeline breakers fall. Otherwise the ideas are the same as on CPUs. "There are no new ideas in databases, but the hardware keeps changing."
