> NOTE: This file is an excerpt of `docs/effective_java.md` bundled into the skill for portability.

# 10. CONCURRENCY

## 66. Synchronize access to shared mutable data

Synchronization prevent a thread from observing an object in an inconsistent state.
Synchronization ensures that each thread entering a synchronized method or block sees the effects
of all previous modifications that were guarded by the same lock.

In Java reading or writing a variable is atomic unless type long or double, but for all atomic operations it does not guarantee that a value written by one thread will be visible to another.

Synchronization is required for reliable communication between threads as well as for mutual exclusion.

```java

	// Broken! - How long would you expect this program to run?
	public class StopThread {
		private static boolean stopRequested;

		public static void main(String[] args) throws InterruptedException {
			Thread backgroundThread = new Thread(new Runnable() {
				public void run() {
					int i = 0;
					while (!stopRequested)
						i++;
				}
			});
			backgroundThread.start();

			TimeUnit.SECONDS.sleep(1);
			stopRequested = true;
		}
	}
```

Because of _hoisting_ the while loop is translated to this:

```java

	if (!done)
		while (true)
			i++;

```

and therefore the loop never stops.

```java

	// Properly synchronized cooperative thread termination
	public class StopThread {
		private static boolean stopRequested;

		private static synchronized void requestStop() {
			stopRequested = true;
		}
		private static synchronized boolean stopRequested() {
			return stopRequested;
		}

		public static void main(String[] args) throws InterruptedException {
			Thread backgroundThread = new Thread(new Runnable() {
				public void run() {
					int i = 0;
					while (!stopRequested())
						i++;
				}
		});
		backgroundThread.start();

		TimeUnit.SECONDS.sleep(1);
		requestStop();
		}
	}
```

Synchronization has no effect unless both read and write operations are synchronized.

Other solution is using _volatile_ modifier, it performs no mutual exclusion, but it guarantees that any thread that reads the field will see the most recently written value.

```java

	public class StopThread {
		private static volatile boolean stopRequested;

		public static void main(String[] args) throws InterruptedException {
			Thread backgroundThread = new Thread(new Runnable() {
				public void run() {
					int i = 0;
					while (!stopRequested)
						i++;
			}
		});
		backgroundThread.start();

		TimeUnit.SECONDS.sleep(1);
		stopRequested = true;
		}
	}
```

Be careful with _volatile_ when using non atomic functions like ++

```java

	private static volatile int nextSerialNumber = 0;

	public static int generateSerialNumber() {
		return nextSerialNumber++;
	}
```

We can synchronize the access to nextSerialNumber and remove _volatile_ or use AtomicLong.
AtomicLong can help us with the synchronization of long values

```java

	private static final AtomicLong nextSerialNum = new AtomicLong();

	public static long generateSerialNumber() {
		return nextSerialNum.getAndIncrement();
	}
```

**effectively immutable**: data object modified by one thread to modify shared it with other threads, synchronizing only the act of sharing the object reference. Other threads can then read the object without further synchronization, so long as it isn't modified again.

**safe publication**: Transferring such an object reference from one thread to others.

_In general:_ When multiple threads share mutable data, each thread that reads or writes the data must perform synchronization

_Best thing to do:_ **Not share mutable data.**

## 67. Avoid excessive synchronization

Inside a synchronized region, do not invoke a method (_alien_) that is designed to be overridden, or one provided by a client in the form of a function object ([Item 21](#21-use-function-objects-to-represent-strategies)). Calling it from a synchronized region can cause exceptions,
deadlocks, or data corruption.
Move alien method invocations out of synchronized blocks. Taking a “snapshot” of the object that can then be safely traversed without a lock.

```java

	// Alien method moved outside of synchronized block - open calls
	private void notifyElementAdded(E element) {
		List<SetObserver<E>> snapshot = null;
		synchronized(observers) {
			snapshot = new ArrayList<SetObserver<E>>(observers);
		}
		for (SetObserver<E> observer : snapshot)
			observer.added(this, element);
	}
```

Or use a _concurrent collection_ ([Item 69](#69-prefer-concurrency-utilities-to-wait-and-notify)) known as CopyOnWriteArrayList. It is a variant of ArrayList in which all write operations are implemented by making a fresh copy of the entire underlying array.
The internal array is never modified and iteration requires no locking.

**open call**: An alien method invoked outside of a synchronized region

_As Rule_:

- **do as little work as possible inside synchronized regions**
- **limit the amount of work that you do from within synchronized regions**

## 68. Prefer executors and tasks to threads

Creating a work queue:

```java

	ExecutorService executor = Executors.newSingleThreadExecutor();
```

Submit a runnable for execution:

```java

	executor.execute(runnable);
```

Terminate gracefully the executor

```java

	executor.shutdown();
```

ExecutorService possibilities:

- wait for a particular task to complete: `background thread SetObserver`
- wait for any or all of a collection of tasks to complete: `invokeAny` or `invokeAll`
- wait for the executor service's graceful termination to complete: `awaitTermination`
- retrieve the results of tasks one by one as they complete: `ExecutorCompletionService`
  \*...

For more than one thread use a _thread pool_. Follow the
[thread-pool guide](thread-pools.md) when choosing and configuring it: restrict cached pools to
tests or controlled low-load workloads, and configure fixed-pool queue capacity explicitly with
`ThreadPoolExecutor` instead of accepting an effectively unbounded queue.

**executor service**: mechanism for executing tasks

**task**: unit of work. Two types.

- Runnable
- Callable, similar to Runnable but returns a value

## 69. Prefer concurrency utilities to _wait_ and _notify_

Given the difficulty of using wait and notify correctly, you should use the higher-level concurrency utilities instead.

- Executor Framework ([Item 68](#68-prefer-executors-and-tasks-to-threads))
- Concurrent Collections
- Synchronizers

**Concurrent Collections**: High-performance concurrent implementations of standard collection interfaces (List, Queue, and Map)  
Use concurrent collections in preference to externally synchronized collections  
Some interfaces have been extended with blocking operations, which wait (or block) until they can be successfully performed. This allows blocking queues to be used for work queues ( _producer-consumer queues_). One or more producer threads enqueue work items and from which one or more consumer threads dequeue and process
items as they become available. ExecutorService implementations, including ThreadPoolExecutor, use a BlockingQueue ([Item 68](#68-prefer-executors-and-tasks-to-threads)).

**Synchronizers**: objects that enable threads to wait for one another, allowing them to coordinate their activities (CountDownLatch, Semaphore, CyclicBarrier, Exchanger)

**wait**: Always use the wait loop idiom to invoke the wait method; never invoke it outside of a loop. The loop serves to test the condition before and after waiting.

```java

	// The standard idiom for using the wait method
	synchronized (obj) {
		while (<condition does not hold>){
			obj.wait(); // (Releases lock, and reacquires on wakeup)
		}
		... // Perform action appropriate to condition
	}
```

**notify**: Wakes a single waiting thread, assuming such a thread exists.

**notifyAll**: Wakes all waiting threads.

Use always use _notifyAll_ (and not forget to use the wait loop explained before)
You may wake some other threads, but these threads will check the condition for which they're waiting and, finding it false, will continue waiting.

**There is seldom, if ever, a reason to use wait and notify in new code.** Use higher-level language

## 70. Document thread safety

Looking for the synchronized modifier in a method declaration is an implementation detail.
To enable safe concurrent use, a class must clearly document what level of thread safety it supports.

- immutable: No external synchronization is necessary (i.e. String, Long, BigInteger)
- unconditionally thread-safe: mutable but with internal synchronization. No need for external synchronization (i.e. Random, ConcurrentHashMap)
- conditionally thread-safe: some methods require external synchronization.(i.e. Collections.synchronized wrappers)
- not thread-safe: external synchronization needed (i.e. ArrayList, HashMap.)
- thread-hostile: not safe for concurrent use (i.e. System.runFinalizersOnExit)

Thread safety annotations are Immutable, ThreadSafe, and NotThreadSafe.
To document a conditionally thread-safe class indicate which invocation sequences require external synchronization, and which lock must be acquired to execute these sequences.

Use private lock object idiom to prevent users to hold the lock for a long period of time in unconditionally thread-safe classes.

```java

	// Private lock object idiom - thwarts denial-of-service attack
	private final Object lock = new Object();

	public void foo() {
		synchronized(lock) {
			...
		}
	}
```

## 71. Use lazy initialization judiciously

Use it if a field is accessed only on a fraction of the instances of a class and it is costly to initialize the field.  
It decreases the cost of initializing a class or creating an instance, but increase the cost of accessing it.  
For multiple threads, lazy initialization is tricky.

```java

	// Normal initialization of an instance field
	private final FieldType field = computeFieldValue();

```

To break an initialization circularity: **synchronized accessor**

```java

	// Lazy initialization of instance field - synchronized accessor
	private FieldType field;
	synchronized FieldType getField() {
		if (field == null)
			field = computeFieldValue();
		return field;
	}
```

For performance on a static field: **lazy initialization holder class idiom**, adds practically nothing to the cost of access.

```java

	// Lazy initialization holder class idiom for static fields
	private static class FieldHolder {
		static final FieldType field = computeFieldValue();
	}
	static FieldType getField() { return FieldHolder.field; }
```

For performance on an instance field: **double-check idiom**.

```java

	// Double-check idiom for lazy initialization of instance fields
	private volatile FieldType field;
	FieldType getField() {
		FieldType result = field;
		if (result == null) { // First check (no locking)
			synchronized(this) {
				result = field;
				if (result == null) // Second check (with locking)
					field = result = computeFieldValue();
			}
		}
	return result;
	}
```

Instance field that can tolerate repeated initialization: **single-check idiom.**

```java

	// Single-check idiom - can cause repeated initialization!
	private volatile FieldType field;
	private FieldType getField() {
		FieldType result = field;
		if (result == null)
			field = result = computeFieldValue();
		return result;
	}
```

## 72. Don't depend on thread scheduler

Thread scheduler determines which runnable, get to run, and for how long.Operating systems will try to make this
determination fairly, but the policy can vary. So any program that relies on the thread scheduler for correctness or performance is likely to be non portable.  
To ensure that the average number of runnable threads is not significantly greater than the number of processors.
Threads should not run if they aren't doing useful work,
tasks should be:

- reasonably small but not too small or dispatching overhead
- independent of one another
- not implement busy-wait

## 73. Avoid thread groups

Thread groups are obsolete.

