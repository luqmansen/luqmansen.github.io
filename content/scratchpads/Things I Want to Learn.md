
And to some extend, want to implement it on my toy rdbms project

In no particular order

 Rust
- Advance Rust Typesystem
	- https://github.com/skyzh/type-exercise-in-rust
	- https://github.com/lilyyy411/rust-type-fuckery
- Async Runtimes 

Storage Engine
- Object store based LSM
	- https://arxiv.org/abs/1812.07527

Query Parsing
- RDP / TDP (rudimentary)
- ANTLR
	- https://tomassetti.me/antlr-mega-tutorial/

SQL Feature
- CTE support
- ASOF join
	- https://questdb.com/blog/asof-join/

Query Optimization Techniques
- Cost based optimizer
	- (too broad, will add more details)
- Query unnesting
	- https://15799.courses.cs.cmu.edu/spring2025/papers/11-unnesting/neumann-btw2015.pdf

Query Execution
- Vectorized Execution
- JIT Query Compilation (not sure where this belongs)

Formal verification
- https://queue.acm.org/detail.cfm?id=3819084
- https://github.com/stonysystems/tla-rs

IO-techniques
- Async-IO (a la tokio)
	- https://duckdb.org/2026/07/31/asynchronous-io
- io_uring
- direct-io

Memory Management
- Region-based memory (Arena)
