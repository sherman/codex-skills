# Sources and Provenance

## Effective Java principles

The skill retains the concurrency principles summarized by Effective Java,
2nd edition Items 66-73 and updated in 3rd edition Items 78-84. The operational
guidance is intentionally organized by current Java concepts rather than book
item numbers.

The removed legacy summary originated from
[HugoMatilla/Effective-JAVA-Summary](https://github.com/HugoMatilla/Effective-JAVA-Summary).
Use the books for the complete discussion; do not treat old snippets or item
numbers as current JDK API guidance.

## Primary Java references

- [Java Language Specification: Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [`java.util.concurrent`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [`ThreadPoolExecutor`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
- [`ExecutorService`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html)
- [Oracle Virtual Threads guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [JEP 525: Structured Concurrency](https://openjdk.org/jeps/525)

## Additional source for bundled thread-pool guidance

The thread-pool sizing heuristics cite Java Concurrency in Practice, chapter 8.
Treat formulas as initial estimates and validate them with current workload
measurements and downstream limits.
