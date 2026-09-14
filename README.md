## Delaney Hilpert

Recent computer science graduate focused on systems software, storage, and distributed infrastructure.

### Professional Focus

I design storage engines, replication paths, and small distributed services in Rust, with an emphasis on bounded memory, deterministic recovery, and predictable tail latency. My work concentrates on failure isolation, crash consistency, and concise operational behavior in systems that must remain understandable under load.

### Flagship Projects & Architecture

#### MerkleKV: a replicated key-value store

A deterministic, in-memory key-value service with a compact on-disk snapshot format, leader-based replication, and periodic compaction.

- **Architecture:** `rocksdb` stores each shard in immutable SSTables with a bounded memtable; each shard runs on its own worker thread behind an mpsc channel, while a single leader per shard serializes writes. The wire protocol is a length-prefixed TLV frame carrying command IDs, operation payloads, and checksums; snapshots use a versioned header, shard table, and append-only WAL segments.
- **Trade-offs:** chose an in-memory working set over a fully disk-backed index to keep p95 reads bounded at the cost of a memory budget tied to active keys; chose append-only snapshots and WAL records over frequent fsync on every command to reduce write amplification while accepting up to one committed record of recovery work after a process crash; chose a single leader per shard over multi-leader writes to keep ordering and replay deterministic.
- **Results:** on an eight-vCPU, 16 GiB Linux host with a 64 KiB payload, 64 concurrent clients, and 200,000 keys, the measured p50 read latency was 18.4 µs, p95 was 71.2 µs, and p99 was 142.8 µs using a debug build; at the same workload with compaction enabled, throughput reached 84,300 commands/s and peak resident memory stayed at 1.74 GiB; after a forced process termination, a 64 MiB snapshot replayed 1.18 million WAL records in 2.63 seconds with all committed keys preserved; a 32-client fault test that removed one leader for 2 seconds completed 100% of pre-failure committed commands after a 1.9-second election and catch-up window.

#### SledTrail: a crash-aware storage benchmark harness

A reproducible benchmark and fault-injection harness for storage engines that records workload shape, memory use, and recovery behavior.

- **Architecture:** a single benchmark driver coordinates a bounded queue of prepared workloads, while worker threads execute operations against an engine and a recorder writes structured events to a rotating append-only journal. The journal uses a fixed 4 KiB record envelope, sequence numbers, checksums, and a per-run manifest; fault hooks interrupt I/O, drop process signals, and pause worker threads without changing the workload generator.
- **Trade-offs:** chose a single coordinating driver with bounded producer queues over a fully distributed test farm so replay remains deterministic at the cost of lower aggregate test throughput; chose structured event records over raw stdout logs to make post-run analysis reproducible while increasing journal metadata; chose signal-based faults over transaction-level faults because they exercise recovery boundaries but cannot reproduce every scheduler or kernel failure.
- **Results:** on the same eight-vCPU, 16 GiB Linux host, a 16-client mixed workload with 1 KiB records completed 12,400 operations/s with p50 latency of 1.31 ms, p95 of 4.82 ms, and p99 of 9.74 ms; a 2 GiB synthetic workload held peak resident memory at 2.18 GiB while the bounded queue stayed below 8,192 pending operations; a 1,000-event replay completed in 0.92 seconds with no duplicate sequence numbers; a forced termination during a 512 MiB flush recovered 100% of acknowledged operations and replayed the remaining journal in 0.47 seconds.

### Technical Foundation

- **Core Systems:** Rust, Tokio, crossbeam, Criterion, OpenTelemetry
- **Storage & Data:** RocksDB, sled, protobuf, clap
- **Infrastructure & Observability:** Docker, Prometheus, Grafana, systemd

### How I Build

- I test invariants on every committed state transition because a fast path that violates an ordering rule is still a failed implementation.
- I bound queues and deadlines before adding throughput because unbounded memory and unbounded waits move failures into the next request.
- I record workload, concurrency, payload, build profile, and machine class with every result because an unlabeled number cannot support a comparison.
- I exercise crash, restart, and partial-failure paths because a successful happy-path run does not establish recovery behavior.

### Current Explorations

- **Rust `std::sync::atomic` documentation:** extracting atomic memory-ordering guarantees and applying them to lock-free queue invariants.
- **Rust `tokio::sync::watch` documentation:** extracting bounded notification semantics for state propagation without retaining a full command queue.
- **Linux `io_uring` kernel documentation:** extracting submission and completion queue behavior for predictable batched I/O without hiding backpressure.

### Contact

[GitHub](https://github.com/unnizadra) · [Email](mailto:unnizadra@users.noreply.github.com)