---
name: effective-java-concurrency
description: "Apply Effective Java (2nd edition) concurrency and thread-pool practices when writing or reviewing multi-threaded Java code, executors, or ExecutorService configuration: synchronization/visibility, locking, named ThreadFactory use, daemon policy, cached/fixed pool selection, bounded queues, workload-based pool sizing, concurrency utilities, thread-safety documentation, lazy initialization, and scheduler independence."
---

# Effective Java Concurrency and Thread Pools

Use the [Effective Java concurrency reference](references/10-concurrency.md)
(2nd edition, Items 66-73) for the core checklist. Use the
[thread-pool reference](references/thread-pools.md) for construction, queueing,
naming, daemon policy, and sizing.
If another repo skill sets stricter rules, follow the stricter one.

## Quick workflow

1. Identify all shared mutable state and define the locking/visibility policy.
2. Prefer not sharing mutable state at all (immutability, confinement, or safe publication).
3. Prefer high-level concurrency utilities (executors, concurrent collections, synchronizers).
4. For each owned thread pool, classify its workload, name its threads, choose daemon status explicitly, configure pool/queue sizes, and plan clean shutdown.
5. Keep synchronized regions small; avoid calling client/overridable code while holding locks.
6. Document the thread-safety guarantees and any required external locking.

## Checklist by item (66-73)

- 66: Synchronize access to shared mutable data (mutual exclusion + visibility); use `volatile` only for visibility; use atomics for atomic updates.
- 67: Avoid excessive synchronization; do as little work as possible under lock; use open calls (move alien method calls outside locks).
- 68: Prefer executors and tasks to threads; apply the thread-pool checklist below; shut down executors cleanly.
- 69: Prefer concurrency utilities to `wait`/`notify`; use concurrent collections and synchronizers; if you must, use the wait-loop idiom and prefer `notifyAll`.
- 70: Document thread safety (immutable / unconditionally thread-safe / conditionally thread-safe / not thread-safe / thread-hostile); document which lock guards which state.
- 71: Use lazy initialization judiciously; prefer eager init; otherwise use holder idiom (static), synchronized accessors, or double-check idiom with `volatile`.
- 72: Do not depend on the thread scheduler for correctness or performance; avoid busy-wait and oversubscription.
- 73: Avoid thread groups (obsolete).

## Thread-pool checklist

Read the [thread-pool reference](references/thread-pools.md) before creating,
reconfiguring, or reviewing a thread pool.

- Supply a `ThreadFactory` for owned pools and give threads a descriptive, pool-specific name with a sequence number.
- Use daemon threads only for work that may stop at any point without harming the application or operating system. Do not rely on daemon threads to finish I/O or execute required `finally` cleanup.
- Restrict cached pools to tests or controlled, low-load workloads. Do not use them for high load or uncontrolled task submission because the thread count can keep growing.
- Configure fixed-pool queue capacity explicitly with `ThreadPoolExecutor`; do not accept the effectively unbounded queue from `Executors.newFixedThreadPool()` when queued work can accumulate.
- Size pools and queues from measured workload behavior and service limits. Treat the reference formulas as starting estimates, then verify them with profiling and load tests.
- Keep CPU-bound and nonblocking-async pools near the available core count. Allow more threads for blocking I/O according to the measured wait-to-compute ratio.
- Consider a `SynchronousQueue` when downstream capacity already limits concurrency and buffering additional work provides no value.

## Common red flags

- Unsynchronized read/write of shared mutable fields (especially "write without synchronization, read with synchronization" or vice versa).
- `volatile` used for non-atomic compound actions (e.g., `++`).
- Calling callbacks / overridable methods while holding a lock.
- Unbounded thread creation or "new Thread(...)" in library code instead of an executor.
- Anonymous/default pool thread names that make thread dumps hard to attribute.
- Daemon workers performing required I/O or cleanup.
- Cached pools receiving uncontrolled work, or fixed pools using an implicit effectively unbounded queue.
- Pool or queue sizes selected without workload measurements and service constraints.
- `wait()` or `notify()` used outside the standard wait-loop idiom.

## References

- Shared state, synchronization, executors, and thread safety: [Effective Java Items 66-73](references/10-concurrency.md)
- Thread-pool construction and sizing: [Thread pool best practices](references/thread-pools.md)
- Reference provenance and edition notes: [Sources](references/source.md)
