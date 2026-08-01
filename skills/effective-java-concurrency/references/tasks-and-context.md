# Task Lifecycle and Context

## Make results and failures observable

- Choose `execute` or `submit` deliberately. A task submitted with `submit` stores failure in its `Future`; observe, return, aggregate, or explicitly report that future.
- Do not create fire-and-forget tasks whose rejection or failure disappears. Define logging, metrics, retry, or caller-visible failure at the submission boundary.
- Executor rejection can be thrown synchronously before a `Future` or stage is returned. Preserve that behavior when it is the API's overload contract; convert it to a failed stage only when the API promises that all failures are delivered through the stage.
- Pass an explicit executor to `CompletableFuture` async stages when the common pool is not part of the contract.
- Treat the common fork-join pool as shared process-wide capacity. Do not put blocking or long-running work there accidentally.
- Prefer stage composition to `get()` or `join()` inside an async callback. Never block a bounded executor worker on work that must run in the same saturated executor.
- Inspect both normal and exceptional completion. Do not add `exceptionally` handlers that silently convert failures into plausible data.
- Use parallel streams only for measured, splittable, side-effect-free CPU work. Do not use them as an implicit executor for blocking I/O or request-level fan-out.

Submission to an `ExecutorService` safely publishes actions before submission to
the task. Task actions become visible after successful `Future.get()`; do not
invent extra synchronization for that handoff.

## Couple related task lifecycles

Define what happens to sibling tasks when one fails, the caller times out, or
the parent request is cancelled. Propagate a deadline where possible and cancel
work whose result is no longer useful.

Use `StructuredTaskScope` only when the project enables the exact preview API
for its target JDK; its API has changed between releases and remains preview
through JDK 26. Otherwise implement failure, timeout, join, and sibling
cancellation explicitly and test each path.

## Preserve interruption and cancellation

- Treat interruption as a request, not proof that work stopped. Ensure loops and blocking operations respond to it.
- Propagate `InterruptedException` when the API permits. Otherwise restore status with `Thread.currentThread().interrupt()` after required cleanup.
- Do not catch `InterruptedException` and continue normally unless the component deliberately owns the cancellation policy.
- Treat `Future.cancel(true)` and `shutdownNow()` as best-effort interruption. Confirm termination separately when lifecycle requires it.
- Inspect the tasks returned by `shutdownNow()`. If a removed task is also a `Future` exposed to a caller, cancel or otherwise complete it; removing it from the queue alone can leave `Future.get()` blocked forever.
- Remember that `CompletableFuture.cancel(true)` completes the future exceptionally but does not interrupt its underlying computation. Arrange cancellation at the operation or task that owns the work.
- Use bounded waits at external or lifecycle boundaries; define the timeout result rather than waiting forever.

## Review scheduled work

- Choose fixed rate only when wall-clock cadence matters and late executions may catch up. Choose fixed delay when the pause must start after the previous run completes.
- Make periodic failures visible. If one execution throws, subsequent executions are suppressed and the returned `ScheduledFuture` completes exceptionally.
- Retain the `ScheduledFuture` when the component needs per-task cancellation or status.
- Consider `ScheduledThreadPoolExecutor.setRemoveOnCancelPolicy(true)` when many long-delay tasks are cancelled; otherwise cancelled tasks may remain queued until their delay elapses.
- Inspect executor shutdown policies and ownership instead of cancelling every scheduled future mechanically.

## Propagate and clean up context

Identify MDC, tracing, security, locale, transaction, and application
`ThreadLocal` state at every asynchronous boundary.

- Prefer the framework or repository context-propagation wrapper.
- For reusable platform-pool workers, capture the intended context, install it for the task, and restore or clear the previous value in `finally`. Never rely on whichever task last used the worker.
- Do not assume `InheritableThreadLocal` propagates submissions to already-created pool threads.
- Virtual threads support thread locals, but do not use per-thread caches of expensive reusable objects when the application may create very many virtual threads.
- On JDK 25 and above, consider `ScopedValue` for bounded, immutable caller-to-callee context when it fits the project; do not introduce preview-version APIs into older targets.

## Primary references

- [`ExecutorService`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html)
- [`CompletableFuture`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [`ScheduledExecutorService`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ScheduledExecutorService.html)
- [`ScheduledThreadPoolExecutor`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ScheduledThreadPoolExecutor.html)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [JEP 525: Structured Concurrency](https://openjdk.org/jeps/525)
