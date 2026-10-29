# PathLock performance baselines

Run from the repository root:

```sh
cargo bench --locked -p ragfs --bench pathlock_perf -- \
  --mode all --iterations 20 --worker-iterations 100 --workers 1,4,16 \
  --output /tmp/pathlock-perf.csv
```

Use `--mode tree`, `--mode concurrency`, or `--mode batch` independently.
These are manual measurements, with no timing thresholds in CI. No server, model,
Redis service, or additional dependencies are required. All business data, databases,
and subprocess coordination files live in temporary directories and are cleaned up.

The Tree benchmark compares SQLite and filesystem providers on the same real
LocalFS datasets: 10–10,000 flat files, plus 200/1,000 subdirectories with one file
each. Setup and three warmup acquisitions are excluded. Latencies measure
`acquire_tree` only; locks are released between samples. `directory_reads` counts
directory listing calls during measured acquire/release cycles, not individual
metadata system calls. A lightweight atomic counter wraps LocalFS and forwards
native CAS operations. SQLite keeps its database outside the business subtree.

The concurrency benchmark uses 1/4/16 workers on disjoint existing file paths.
Each measured operation uses a different path, so a failed release does not turn
the following samples into logical conflicts on the same retained token.
It exercises `PathLockManager` with `timeout=0`, with no benchmark-side Busy
retries. It compares shared-provider threads (SQLite's single connection/mutex),
independent-provider threads (separate SQLite connections), and separate processes
sharing the same filesystem and database. Each worker warms up three operations
before a start barrier. Each runtime uses four Tokio worker threads, including
each subprocess. Latencies include both acquire and release, including
scheduling, mutex queues, and database waits. Logical lock contention is deliberately
excluded so database contention failures remain visible.

CSV rows include successful operations/second, successful-operation P50/P95/P99,
and failure counts by acquisition/release phase and Timeout/Busy error type.
All workers' latency samples are pooled; quantiles use nearest
rank. Throughput uses the controller's wall time, including barrier/reporting
overhead, especially for subprocesses. Failed operations are counted separately
and excluded from latency percentiles; always check failures before comparing rates.

The benchmark uses SQLite WAL with `synchronous=FULL` and the production 100 ms
Busy timeout per attempt, with three total attempts and 1 ms / 4 ms backoff.
Retries are inside the provider and cover all operations; the benchmark adds none.
It measures warm filesystem caches, not cold storage or application
file deletion/copy time. Twenty Tree samples give only a coarse tail estimate.
Repeat runs on otherwise idle hardware, record OS/CPU/Rust version, and use more
samples for stable P95/P99 comparisons. Thread and process results are different
workloads and should not be interpreted as a pure connection-pool speedup.

The batch benchmark measures 1/5/13 Exact paths of equal depth, with shared or
distinct immediate parents. Directories already exist; three warmup cycles are
excluded. Acquire and release have separate latency rows and filesystem call
totals (`read`, `stat`, `write`, `compare_write`, `compare_remove`, `mkdir`).
Divide totals by `successes` for calls per batch. These are trait-level calls,
not S3 HTTP requests: conditional operations can issue multiple HTTP requests.
Batch throughput uses each phase's cumulative duration, not whole-cycle wall
time. Any batch acquire/release failure aborts the run instead of yielding a
partial comparison. Use at least 200 samples, repeat sequential runs on the
same host, and compare the same harness on unmodified and modified sources.
LocalFS does not predict TOS end-to-end latency.

Native metrics export `lock_phase_duration_seconds` with bounded `phase`,
`outcome`, and `batch_size` labels. Histogram count is phase invocation count;
sum measures total native phase time. `acquire_total` includes retries, owner
waits, I/O and registry work; `release_total` includes owner serialization and
owned-token removal. Totals overlap their constituent phases; do not sum all
series. Outcomes include failed and cancelled work. No paths or capabilities
are labels. Per-phase recording itself adds CPU overhead, measurable in the
one-path LocalFS case. Python cProfile still measures the surrounding workflow;
use these histograms to split the native wait and existing filesystem operation
metrics to correlate backend I/O.
