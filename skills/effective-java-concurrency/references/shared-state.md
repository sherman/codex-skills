# Shared State and Synchronization

## Establish the contract

Before changing code, identify:

- which state is shared and which invariant spans multiple fields or objects;
- who writes, who reads, and how the object reference is published;
- which lock, atomic variable, confinement boundary, or concurrent collection provides the required ordering;
- whether collaborators called concurrently are themselves safe for concurrent use.

Thread safety is a property of the whole access protocol. Synchronizing only a
writer or only a reader does not establish a consistent protocol.

## Prefer non-sharing

Prefer these options in order when they fit the design:

1. immutable values with final fields and no constructor escape;
2. thread confinement or ownership transfer;
3. immutable snapshots or messages;
4. safe publication through class initialization, a volatile reference, the same lock, a concurrent collection, or a documented concurrency utility;
5. shared mutation guarded by one explicit policy.

A final reference does not make the referenced object immutable. Safe
publication does not make later unsynchronized mutation safe.

## Choose the synchronization mechanism

- Use `synchronized` or `Lock` when one invariant spans multiple operations or fields.
- Prefer `synchronized` for simple lexical mutual exclusion. Use `Lock` when timed or interruptible acquisition, multiple conditions, or non-lexical locking is required, and always unlock in `finally`.
- Use atomics for independent atomic state transitions. Do not assume a sequence across several atomics is atomic.
- Use `volatile` for visibility and ordering when the update itself is already atomic and no multi-variable invariant is involved. `volatile int count; count++` is still a read-modify-write race.
- Use concurrent collections and their atomic methods such as `compute`, `putIfAbsent`, or conditional `remove` for compound collection actions.
- Prefer `CountDownLatch`, `Semaphore`, `Phaser`, blocking queues, or futures to handwritten `wait`/`notify` protocols.

Do not mix synchronized and unsynchronized access to state participating in the
same invariant.

## Apply lock discipline

- Prefer a stable private lock for externally accessible mutable classes. Do not lock on interned strings, class literals shared with unrelated code, or publicly reachable objects.
- Keep critical sections small, but do not split an atomic invariant merely to shorten a block.
- Move callbacks, overridable methods, logging hooks, network calls, and other unknown or blocking work outside the lock. Snapshot required state first when possible.
- Avoid nested locks. When unavoidable, define and follow one global lock order on every path.
- Recheck conditions in a loop after `wait()` or `Condition.await()` because wakeups do not prove the condition. Prefer higher-level utilities in new code.
- Never rely on sleep duration, thread priority, or scheduler fairness for correctness.

## Publish and initialize safely

Prefer eager initialization. For a lazy static value, use class initialization:

```java
private static final class Holder {
    static final Service INSTANCE = createService();
}

static Service service() {
    return Holder.INSTANCE;
}
```

For a lazy instance value, prefer a synchronized accessor unless profiling
justifies double-checked locking. If double-checked locking is necessary, make
the field `volatile` and use a local variable:

```java
private final Object initLock = new Object();
private volatile Service service;

Service service() {
    Service result = service;
    if (result == null) {
        synchronized (initLock) {
            result = service;
            if (result == null) {
                service = result = createService();
            }
        }
    }
    return result;
}
```

Do not let `this` escape from a constructor to a thread, callback, registry, or
executor before construction finishes.

## Document thread safety

State whether a type is immutable, thread-safe, conditionally thread-safe, or
not thread-safe. For conditional safety, name the required lock and the method
sequences it protects. Document executor and thread affinity, safe-publication
requirements, and whether callbacks may run concurrently. Use project-provided
thread-safety annotations only when their dependency and semantics are known.

## Primary references

- [JLS 17.4: Memory Model](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html#jls-17.4)
- [`java.util.concurrent` memory-consistency properties](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html#MemoryVisibility)
