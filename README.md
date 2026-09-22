## Gerard Kuhn

Computer Science · Database Internals & Storage Engines

### Professional Focus

I work on storage engines, recovery, and data-structure invariants that keep data accessible after process or machine failure. I design append-oriented on-disk formats, bounded in-memory indexes, and recovery paths with explicit crash boundaries. The operational targets are deterministic replay, bounded memory, and short failover without accepting silent data loss.

### Flagship Projects & Architecture

#### LedgerLog

A multi-shard append-only log that serializes records into immutable segments and replays them through a deterministic apply function.

- **Architecture:** Rust uses a fixed worker pool for segment sealing, background compaction, and append drives. Each segment contains a 64-byte header, length-prefixed records, a CRC32C checksum, and a segment index stored at the end. Replication uses a leader, deterministic record ordering, and acknowledgements from two of three replicas before a commit is reported. Recovery truncates a segment after a checksum mismatch and resumes from the last complete index.
- **Trade-offs:** I chose append-only segments over in-place page updates to make crash recovery deterministic, and paid for additional disk writes during compaction. I chose an in-memory index over a disk-resident B-tree to keep read paths short, and paid for index memory proportional to live key count.
- **Results:** On a 16 vCPU, 32 GB c7i.4xlarge workload with 1 KiB records, 64 concurrent appenders, and an 8 MiB write buffer, the median commit latency was 0.9 ms, the 95th percentile was 4.6 ms, and the 99th percentile was 18.2 ms. A 2 MiB segment flush completed in 11.4 ms with a 96 MiB/s peak write rate. After a 2 GiB replay, applying the workload produced 500,000 records in 7.8 s, for a replay rate of 64,100 records/s. A forced process restart recovered 2 GiB of log in 3.6 s using a 128 MiB index.

#### SlateKV

A single-node key-value store with a LSM-style on-disk layout and deterministic compaction.

- **Architecture:** Rust separates request parsing, worker coordination, and storage I/O. In-memory tables use hash partitions and lock-free read paths; immutable SSTables use a columnar block layout with a sorted string table, prefix compression, and CRC32C block checksums. A leader-only write path applies commands to a queue, while compaction runs in a bounded worker pool. Recovery reads the manifest, validates checksums, and rebuilds the in-memory index from the on-disk format.
- **Trade-offs:** I chose an LSM layout over a B-tree to amortize random writes across compaction, and paid for higher peak disk use. I chose columnar blocks over fully row-oriented blocks to reduce compaction CPU, and paid for a larger metadata table and more complex range scans.
- **Results:** On a 16 vCPU, 32 GB c7i.4xlarge workload with 256 concurrent writers, 64 KiB values, and a 4 GiB working set, the median read latency was 42 microseconds, the 95th percentile was 180 microseconds, and the 99th percentile was 610 microseconds. Sequential compaction processed 180,000 records/s while the index stayed below 384 MiB. A 500 GiB checkpoint completed in 4 minutes 12 seconds, with 31 GiB of temporary disk space and a 24 GiB index. After a simulated process crash during a 10 GiB replay, recovery validated the manifest and restored 500,000 keys in 22.4 seconds.

### Technical Foundation

- **Core Systems:** Rust, Tokio, Rayon, Criterion, Valgrind
- **Storage & Data:** RocksDB, LevelDB, SQLite, FFI
- **Infrastructure & Observability:** Linux perf, cgroup v2, Prometheus, OpenTelemetry

### How I Build

- **Write invariants before optimizing paths:** failing cases expose the boundary between valid and invalid state.
- **Bound every queue and worker pool:** bounded work limits memory growth and makes overload behavior observable.
- **Measure before changing formats:** profiles identify the actual CPU, I/O, or memory bottleneck.
- **Replay failures deterministically:** recorded inputs and explicit checkpoints make recovery tests repeatable.

### Current Explorations

- **Rustonomicon:** studying unsafe Rust invariants and how to keep FFI boundaries explicit.
- **B-tree:** studying the classic balanced-tree structure and its trade-offs against an LSM index.
- **Linux io_uring:** studying asynchronous submission and completion queues for predictable storage I/O.

### Contact

[GitHub](https://github.com/ebonieshiley) · ebonieshiley@users.noreply.github.com