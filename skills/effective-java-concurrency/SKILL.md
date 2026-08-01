---
name: effective-java-concurrency
description: "Review, design, and implement Java concurrency involving shared mutable state, synchronization, executors, futures, schedulers, platform threads, or virtual threads. Use for code changes and reviews where thread safety, memory visibility, cancellation, context propagation, executor ownership, overload behavior, lifecycle, or measured concurrency performance matters. Do not use for basic Java explanations without a concrete concurrent code path."
---

# Java Concurrency Review and Implementation

Follow project-local instructions, the target JDK, and framework ownership and
lifecycle contracts before this guidance. Inspect existing concurrency helpers
and call sites before introducing a new abstraction. When contracts conflict,
follow the project contract and explain the conflict.

## Workflow

1. Inspect the build configuration for the target JDK and preview features, then identify the runtime or framework that creates and manages threads.
2. Map the concurrent path: entry threads, submitted tasks, workers, shared state, downstream limits, task results, cancellation, context, and shutdown owner.
3. State the invariant and ownership model before choosing synchronization. Prefer immutability, confinement, safe publication, and high-level concurrency utilities.
4. Classify the workload before selecting platform-thread pools, virtual threads, or an existing async execution model. Do not migrate execution models without a requested or measured benefit.
5. Analyze steady state, saturation, failure, cancellation, interruption, and shutdown. Do not review only the successful path.
6. Make the narrowest target-JDK-compatible change and document externally visible thread-safety and lifecycle guarantees.
7. Verify correctness deterministically; measure throughput changes under representative load.

## Route the task

Load only the references needed for the concrete path:

- Read [Shared state and synchronization](references/shared-state.md) for visibility, atomicity, locking, publication, lazy initialization, concurrent collections, or thread-safety contracts.
- Read [Executors and virtual threads](references/thread-pools.md) for executor construction, queueing, saturation, rejection, sizing, virtual-thread selection, or shutdown.
- Read [Task lifecycle and context](references/tasks-and-context.md) for `Future`, `CompletableFuture`, scheduled work, failure, cancellation, interruption, `ThreadLocal`, MDC, security, or tracing context.
- Read [Review and verification](references/review-and-testing.md) for code review, diagnostics, stress testing, or performance work.

## Review discipline

- Trace an actual concurrent path and establish the violated invariant, happens-before relation, ownership rule, or capacity contract before reporting a finding.
- Report the exact location, triggering interleaving or state, consequence, and narrow remedy. Distinguish correctness defects from throughput risks and optional hardening.
- Treat search matches as leads, not findings. Check framework defaults, injected-resource ownership, target-JDK behavior, and downstream admission controls.
- Do not infer an API promise solely from its return type. For example, returning `CompletionStage` does not by itself prove that executor rejection must be encoded as a failed stage rather than thrown synchronously.
- Avoid blanket findings. An unbounded queue can be intentional, a missing virtual-thread name is not inherently a defect, and an extra semaphore is not automatically useful when an existing resource already limits concurrency.

## Implementation discipline

- Guard every access participating in a shared invariant with one consistent policy. Use `volatile` for visibility, not compound atomicity.
- Keep lock scope small, use a stable private lock where practical, and do not invoke unknown callbacks or blocking I/O while holding it.
- Treat interruption as cooperative cancellation: propagate it or restore interrupt status when an API boundary cannot throw it.
- Shut down only executors the component owns. Prefer a repository lifecycle helper such as `ExecutorShutdownAgent` only after checking its implementation and existing call sites; a helper name is not proof that its contract fits. Otherwise use the JDK two-phase shutdown pattern.
- When forced shutdown removes queued tasks, trace caller-visible `Future` objects and ensure they reach a terminal state instead of leaving `get()` blocked forever.
- Preserve the established thread-safety, exception, rejection, context-propagation, and lifecycle contracts unless the task explicitly changes them.

## Deliver the result

For a review, list actionable findings first in severity order. Include the
concurrent path and evidence for each finding, then state verification gaps. If
no contract is violated, say so instead of manufacturing a recommendation.

For an implementation, include focused tests or measurements and state any
target-JDK, ownership, or workload assumption that materially affects the
design.

## Sources

Use [Sources and provenance](references/source.md) only when checking editions
or primary documentation.
