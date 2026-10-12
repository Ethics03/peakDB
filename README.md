# PeakDB

PeakDB is an embedded analytical database written in Rust, built from first
principles and eventually optimized for querying agent execution traces and
evaluation workloads.

The engine is built around columnar data, vectorized execution, relational
operators, query planning, and optimization. These foundations will be
specialized using real agent workloads and measured bottlenecks.

## Project direction

PeakDB will evolve in three stages:

1. A focused vectorized OLAP engine.
2. A benchmarked Rust query execution engine.
3. An embedded analytical engine specialized for agent traces, evaluations,
   cost analysis, and execution provenance.

The guiding principle is:

> Build general relational foundations, then specialize using real agent
> workloads and measured bottlenecks.

Potential specializations must demonstrate a concrete benefit such as lower
query latency, reduced memory use, faster ingestion, or better support for
replay and provenance. Performance claims will be evaluated against established
engines such as DuckDB and Apache DataFusion on clearly defined workloads.

## Status

PeakDB is in early development. Current work is focused on the core data model,
vectorized execution primitives, and relational operators.

See [Database Engine Basics](docs/database-engine-basics.md) for the underlying
design concepts.
