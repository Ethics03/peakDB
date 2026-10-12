# Database Engine Basics for a Rust OLAP Project

This guide establishes the concepts needed to build a small analytical database engine inspired by DuckDB. The initial goal is not full SQL compatibility or durable storage. It is to understand how tabular data moves through expressions and relational operators to produce a result efficiently.

At its core, an analytical database performs one transformation:

```text
Tabular data + query
        |
        v
Relational operations
        |
        v
Result table
```

## 1. Tables and schemas

Consider an `orders` table:

| order_id | region | amount |
| ---: | --- | ---: |
| 1 | India | 500 |
| 2 | UK | 300 |
| 3 | India | 700 |

Its schema describes the columns and their types:

```text
order_id: INTEGER
region:   STRING
amount:   INTEGER
```

A row is one record:

```text
(1, "India", 500)
```

A column contains values with the same meaning and usually the same physical type:

```text
amount = [500, 300, 700]
```

Types determine which operations are valid and how values are represented in memory. Adding two integers is straightforward; adding an integer and a string is not. A fixed-width integer can be stored directly, while a string normally requires offsets or references to separately stored bytes.

## 2. Missing values and SQL NULL

SQL distinguishes a missing value from an ordinary value:

```text
amount = [500, NULL, 700]
```

A columnar engine can represent this with values and a validity mask:

```text
values:   [500, 0, 700]
validity: [  1, 0,   1]
```

The value at a null position is ignored. The validity bit indicates whether that position contains a meaningful value.

`NULL` also introduces three-valued logic:

```sql
NULL = 5       -- unknown
NULL = NULL    -- unknown
NULL IS NULL   -- true
```

A SQL filter retains rows for which its condition is `TRUE`. It rejects both `FALSE` and `UNKNOWN`.

Null handling appears in nearly every execution kernel. A common optimization is to have a fast path for columns with no nulls and a separate path that checks the validity mask.

## 3. Declarative queries

An imperative program specifies how to perform a calculation:

```rust
let mut totals = HashMap::new();

for order in orders {
    if order.amount > 100 {
        *totals.entry(order.region).or_insert(0) += order.amount;
    }
}
```

SQL describes the desired result:

```sql
SELECT region, SUM(amount)
FROM orders
WHERE amount > 100
GROUP BY region;
```

The SQL query does not dictate which columns to read first, which hash-table implementation to use, or how many threads to run. The database chooses an execution strategy.

This separation is fundamental:

```text
Query meaning != execution strategy
```

## 4. Relational operators

Most analytical queries can be decomposed into a small set of operators.

### Scan

A scan reads data from a table or file:

```text
Scan(orders)
```

### Filter

A filter, also called a selection, retains matching rows:

```sql
WHERE amount > 100
```

```text
Input:  [(India, 500), (UK, 50), (US, 300)]
Output: [(India, 500),           (US, 300)]
```

### Projection

A projection chooses or computes columns:

```sql
SELECT region, amount * 1.18
```

```text
Input:  region, amount, customer, date
Output: region, amount_with_tax
```

A filter chooses rows. A projection chooses or calculates columns.

### Aggregation

An aggregation combines many input rows:

```sql
SELECT region, SUM(amount)
FROM orders
GROUP BY region;
```

```text
Input:                 Output:
India, 500             India, 1200
UK,    300             UK,     300
India, 700
```

### Join

A join combines related rows from two inputs:

```sql
SELECT orders.amount, customers.name
FROM orders
JOIN customers
  ON orders.customer_id = customers.id;
```

An order with `customer_id = 42` is matched with the customer whose `id = 42`.

### Sort and limit

A sort arranges rows:

```sql
ORDER BY amount DESC
```

A limit retains only part of the output:

```sql
LIMIT 10
```

Scan, filter, projection, aggregation, join, sort, and limit form the basic vocabulary of a query engine.

## 5. Operator trees

Consider this query:

```sql
SELECT region, SUM(amount)
FROM orders
WHERE amount > 100
GROUP BY region
ORDER BY SUM(amount) DESC;
```

Its operator tree is approximately:

```text
Sort by sum descending
          ^
Aggregate by region
          ^
Filter amount > 100
          ^
Scan orders
```

Plans are normally read from bottom to top:

1. Scan `orders`.
2. Retain orders with `amount > 100`.
3. Group the remaining rows by `region`.
4. Calculate each group's sum.
5. Sort the groups by the calculated sum.

The output of every operator is conceptually another table, even when the implementation streams small batches rather than materializing whole intermediate tables.

## 6. Logical and physical plans

A logical plan describes the meaning of a query:

```text
LogicalScan
LogicalFilter(amount > 100)
LogicalAggregate(region, SUM(amount))
LogicalSort(sum DESC)
```

A physical plan selects concrete algorithms:

```text
ColumnScan
VectorizedFilter
HashAggregate
RowBasedSort
```

One logical operator may have several physical implementations. A logical join could become a nested-loop join, hash join, merge join, or index join. The planner chooses among them based on available properties and estimated cost.

This distinction lets the database improve execution without changing the meaning of SQL.

## 7. Cardinality and selectivity

Cardinality is the number of rows produced by an operator:

```text
Scan:       10,000,000 rows
Filter:        100,000 rows
Aggregation:       200 rows
```

Selectivity is the fraction of input rows that survive a filter:

```text
input rows:  10,000
output rows:    100
selectivity:      1%
```

These quantities affect algorithm choice. A highly selective filter should usually run early so that later operators process less data:

```text
Preferred:
10M rows -> filter -> 100K rows -> join

Potentially expensive:
10M rows -> join -> large intermediate result -> filter
```

Moving a filter closer to its data source is called filter pushdown.

## 8. The shape of an OLAP workload

Online analytical processing workloads typically:

- Read many rows.
- Read only a subset of columns.
- Perform aggregations, joins, and sorting.
- Make relatively few small updates.
- Return much less data than they examine.

A query might consume 500 million sales rows and return revenue totals for 200 countries. The engine therefore needs to minimize the CPU and memory cost paid for each input value.

## 9. Row and column layouts

A row-oriented representation stores complete records together:

```text
[1, India, 500]
[2, UK,    300]
[3, India, 700]
```

A column-oriented representation stores values by column:

```text
order_id: [1, 2, 3]
region:   [India, UK, India]
amount:   [500, 300, 700]
```

For this query:

```sql
SELECT SUM(amount) FROM orders;
```

a column-oriented engine can primarily read the `amount` column. This provides several advantages for analytical work:

- Irrelevant columns need not be read.
- Adjacent values have the same physical type.
- CPU-cache use is more predictable.
- Similar values can often be compressed effectively.
- Values can be processed in tight loops.

Row storage remains attractive for retrieving or modifying complete individual records. Column storage is attractive when scanning a small number of columns across many records.

## 10. Vectorized execution

A row-at-a-time engine repeatedly performs per-row bookkeeping:

```text
Fetch one row
Evaluate its filter
Update the aggregate
Fetch the next row
...
```

A vectorized engine works on batches:

```text
Read a batch of amount values
Evaluate the filter over the batch
Pass matching positions to the aggregate
Repeat
```

A batch can be represented as a data chunk:

```text
DataChunk
|- region vector: [India, UK, India, ...]
|- amount vector: [500, 300, 700, ...]
`- row count: 2048
```

The exact batch size is an implementation choice that should be benchmarked. Vectorization means that one operator invocation processes a batch rather than a single row. SIMD may help some kernels, but it is not the definition of vectorized execution.

Batch processing can improve cache locality, reduce dispatch overhead, and give the compiler better loops to optimize.

## 11. Selection vectors

Suppose a filter receives:

```text
amounts = [500, 20, 700, 40, 300]
```

For `amount > 100`, the matching positions are:

```text
selection = [0, 2, 4]
```

Downstream operators can use these positions without immediately copying all matching values:

```text
amount[0] = 500
amount[2] = 700
amount[4] = 300
```

This avoids materializing a new column after every filter. However, indirect access can become expensive when chunks are sparse, so an engine may eventually compact selected values into a dense chunk.

## 12. Streaming and blocking operators

Streaming operators can produce output as input arrives:

- Filter.
- Projection.
- Limit.
- Many scans.

```text
Input chunk -> filter -> output chunk
```

Blocking operators need to accumulate state before returning most or all of their output:

- Hash aggregation.
- Sorting.
- The build phase of a hash join.
- Many window operations.

A sort, for example, cannot identify the first result until it has considered every input row:

```text
Consume input chunks
        |
        v
Build internal state
        |
        v
Sort
        |
        v
Produce output chunks
```

Blocking operators divide a physical plan into execution pipelines.

## 13. Hash tables in query execution

Hash tables support several important database operations:

- `GROUP BY`.
- Hash joins.
- `DISTINCT`.
- Set operations.
- Some `IN` expressions.

For grouped aggregation:

```text
hash("India") -> aggregate state { sum: 1200 }
hash("UK")    -> aggregate state { sum: 300 }
```

For a hash join:

```text
hash(customer_id) -> matching customer rows
```

The performance of an analytical engine is heavily affected by hash-table layout, hashing cost, allocation behavior, collision handling, and CPU-cache locality.

## 14. Main database subsystems

A complete database contains several major subsystems:

```text
Frontend
|- parser
|- binder
`- type checker

Planner
|- logical planner
|- optimizer
`- physical planner

Execution engine
|- vectors and chunks
|- expressions
|- operators
`- pipelines

Storage engine
|- file format
|- compression
|- buffer manager
`- indexes

Transaction system
|- MVCC
|- write-ahead log
|- commit and rollback
`- crash recovery
```

The Rust project should initially concentrate on this path:

```text
Data representation
        |
        v
Expressions
        |
        v
Operators
        |
        v
Plans
        |
        v
Optimization
```

Persistent storage and transaction processing can be added after the execution model is understood and measurable.

## 15. Core mental model

The essential ideas are:

1. A query transforms one or more tables into another table.
2. SQL describes the desired result rather than the execution procedure.
3. Queries decompose into a small set of relational operators.
4. Logical operations are separate from physical algorithms.
5. OLAP performance depends on processing many values at low per-value cost.
6. Column vectors and data chunks are the natural units of analytical execution.

The next topics should be studied in this order:

1. Relational algebra and SQL semantics.
2. Physical column and vector representation.
3. Expression evaluation over vectors.
4. Scan, filter, and projection operators.
5. Hash aggregation and hash joins.
6. Physical plans and execution pipelines.
7. Rule-based and cost-based optimization.
8. Parallel execution and memory management.
9. Persistent storage and transactions.

## Further reading

- [Overview of DuckDB internals](https://duckdb.org/docs/current/internals/overview)
- [DuckDB execution format](https://duckdb.org/docs/current/internals/vector)
- [Design and Implementation of DuckDB Internals](https://duckdb.org/library/design-and-implementation-of-duckdb-internals/)
