# Review and Verification

## Discover concurrent paths

Start with build files and framework configuration, then search Java sources for
likely boundaries:

```bash
rg -n --glob '*.java' 'new Thread\(|Executors\.new|new ThreadPoolExecutor|CompletableFuture\.(runAsync|supplyAsync)|parallelStream\(|commonPool\(|synchronized|volatile|ThreadLocal|wait\(|notify(All)?\(|shutdown(Now)?\('
```

Also search for the concrete executor bean, field, helper, task type, callback,
and lifecycle hook. Trace callers and call sites before deciding ownership or
thread affinity. A match is not a defect by itself.

For each candidate, record:

1. the submitting or calling thread;
2. the worker and execution model;
3. shared state and its guard or publication edge;
4. downstream capacity and existing admission controls;
5. result, exception, cancellation, and timeout consumers;
6. context propagation and cleanup;
7. executor creator and shutdown owner.

## Write evidence-backed findings

For each finding, provide:

- exact file and line;
- the concrete interleaving, saturation state, or lifecycle event;
- the violated invariant or contract;
- the user-visible consequence;
- the narrowest compatible fix.

Separate correctness failures such as races, deadlocks, lost tasks, or leaked
context from capacity risks and optional diagnostics. Check whether code is
reachable and concurrently invoked before assigning severity.

## Test correctness deterministically

- Use the repository wrapper and its configured target JDK. Compile every changed API path against that target.
- Coordinate tests with `CountDownLatch`, `CyclicBarrier`, `Phaser`, futures, or explicit hooks. Do not use `Thread.sleep` as proof of ordering.
- Put bounded timeouts on tests so failure terminates; do not use timeout duration as synchronization.
- Repeat stress scenarios enough to explore interleavings. Use [jcstress](https://openjdk.org/projects/code-tools/jcstress/) for Java Memory Model outcomes that ordinary unit tests cannot establish reliably.
- Exercise normal completion, exceptional completion, cancellation, interruption, saturation, rejection, and shutdown for changed executor code.
- In forced-shutdown tests, assert that caller-visible futures for removed queued tasks become done or cancelled.
- Verify teardown: owned executors terminate, injected executors remain open, and tests do not leak non-daemon workers.
- Include a negative control or an unaffected path when changing a broad concurrency rule to detect overcorrection.

## Verify performance claims

Measure before and after under a representative arrival pattern. Record at least:

- task rate, service time, queue wait, and end-to-end latency distributions;
- active and maximum platform workers, queue depth and remaining capacity, completed tasks, and rejections;
- downstream pool utilization, acquisition timeouts, and caller timeouts;
- allocation and CPU profiles when changing concurrency primitives or stream usage.

Prefer the repository's existing metrics wrapper. If Micrometer is already in
use, instrument the executor through the established binder or wrapper rather
than adding a second metrics path solely for this review.

Drive the system beyond its intended steady-state capacity to verify overload
behavior. A larger pool or queue can move contention or timeout waste rather
than improve throughput.

For virtual threads, use JFR and thread dumps to inspect blocking behavior and
resource pressure. On JDK 21-23, inspect `jdk.VirtualThreadPinned`; on JDK 24 and
above, `synchronized` no longer causes pinning, but native or foreign-function
frames can still pin carriers. Treat virtual-thread start/end volume as a
diagnostic signal, not a reason to pool virtual threads.

## Report verification honestly

State which focused tests, stress tools, load tests, and runtime diagnostics ran.
Separate failures caused by the patch from unrelated dependency, toolchain,
sandbox, or existing-suite failures. If a concurrency claim remains untested,
name the missing evidence and avoid presenting it as confirmed.
