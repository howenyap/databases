# CMU 15-445 #13: Query Execution Part 1: questions

Answers: [answer.md](answer.md). Interactive version: [quiz.html](quiz.html).

## 1. Which operator is a pipeline breaker, meaning it can't give its parent any output until it has all of its input?

_Around 6:15_

- A) Filter (WHERE predicate)
- B) Projection
- C) Sort
- D) Sequential scan

## 2. In the iterator (Volcano) model, what does a call to next() return?

_Around 12:25_

- A) All of the operator's output at once
- B) One tuple, or null/EOF when there are no more
- C) A batch of about 1024 tuples
- D) A pointer to a buffer the scheduler fills

## 3. The materialization model passes ALL output up at once. What is its main weakness, and what is the main fix?

_Around 21:20_

- A) Too many next() calls; fixed with SIMD
- B) It passes huge intermediate results (a billion tuples for a LIMIT 10); fixed with operator fusion
- C) It can't do joins; fixed with hash tables
- D) It only works on disk; fixed with prefetching

## 4. What workload does Pavlo say suits the materialization model?

_Around 26:15_

- A) OLAP over billions of rows
- B) In-memory OLTP that touches few tuples
- C) Streaming
- D) Approximate queries

## 5. Why is the vectorized (batch) model the standard for OLAP?

_Around 27:20_

- A) It sends the fewest bytes between operators
- B) Fewer next() calls, plus tight branch-free loops over batches that can use SIMD or GPUs
- C) It needs no buffer pool
- D) It makes LIMIT easier than the iterator model does

## 6. In the push-based (bottom-to-top) plan direction, what drives execution?

_Around 33:14_

- A) The root calls next() on its children
- B) A scheduler runs pipelines as tasks in dependency order and tells each one where its input data is
- C) Each leaf polls its parent
- D) The optimizer runs operators inline

## 7. A block's zone map says min=100, max=400. The query is WHERE val > 600. What happens?

_Around 49:38_

- A) Read the block and filter every tuple
- B) Skip the block entirely, because nothing in it can match
- C) Return an approximate count
- D) Use the zone map as an index to find the exact tuples

## 8. A multi-index scan has an AND between two indexed predicates. How are the record IDs combined, and what does Postgres call this?

_Around 54:49_

- A) Union; 'index merge'
- B) Intersection; 'bitmap scan'
- C) Intersection; 'index merge'
- D) Union; 'bitmap scan'

## 9. What causes the Halloween problem?

_Around 60:51_

- A) Queries running past midnight on Oct 31
- B) An update moves a tuple's physical location, so the scan sees the same logical tuple again and updates it twice
- C) Two transactions deleting the same row
- D) A JIT bug in the WHERE clause

## 10. In the live demo, what does Postgres JIT-compile, and how did it compare to DuckDB?

_Around 72:27_

- A) The whole plan, and it beat DuckDB
- B) Only the WHERE-clause expressions; about 4s down to 2.7s, and DuckDB still finished in under 1s
- C) Only joins; both took about 4s
- D) The optimizer; DuckDB was slower

## 11. Computing UPPER('wu-tang') once and reusing the result for every tuple is called what?

_Around 75:26_

- A) Common subexpression elimination
- B) Constant folding
- C) Operator fusion
- D) Late materialization
