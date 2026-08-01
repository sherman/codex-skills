# Thread pool best practices

## Содержание

- [TL;DR](#tldr)
- [Именованные thread pools](#именованные-thread-pools)
- [Потоки-демоны](#потоки-демоны)
- [Cached thread pool](#cached-thread-pool)
- [Fixed thread pool](#fixed-thread-pool)
- [Saturation и rejection policy](#saturation-и-rejection-policy)
- [Завершение executor](#завершение-executor)
- [Размер thread pool](#размер-thread-pool)
- [Virtual threads](#virtual-threads)

## TL;DR
* Всегда передавайте в конструктор создания platform thread pool фабрику [ThreadFactory](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadFactory.html).
* Используйте флаг daemon у потока только в том случае, если поток делает
работу, которая может быть прервана в любое время без каких либо последствий для приложения или операционной системы.
* Используйте cached thread pool там, где не предполагается какой-то
значительной нагрузки, в тестах.
* Не используйте cached thread pool в высоконагруженных приложениях и там, где вы никак не управляете количеством задач, которые могут попасть в thread pool.
* Всегда явно конфигурируйте размер очереди в fixed thread pool.
* Явно выбирайте rejection policy для bounded-очереди и определяйте поведение вызывающего кода при перегрузке.
* Завершайте только те executor, которыми владеет компонент. На обычном timeout
  path после `shutdownNow()` повторно проверяйте termination; на interrupt path
  следуйте documented prompt-return или termination contract и восстанавливайте
  interrupt status.
* Для JDK 21 и выше используйте один virtual thread на blocking-задачу; не объединяйте virtual threads в pool и не применяйте к ним формулу `W/C`.

## Именованные thread pools

### Снипет

Для JDK 21 и выше:

```java
ThreadFactory threadFactory = Thread.ofPlatform()
        .name("CoreExecutor-", 0)
        .daemon(false)
        .factory();
```

Передайте эту фабрику в executor с осознанно выбранными queue и rejection
policy. Полный bounded-пример приведен в разделе про fixed thread pool.

### Мотивация

В thread dump, пулы без имени будут анонимными.
```
"pool-1-thread-1" #13 prio=5 os_prio=0 cpu=0.56ms elapsed=3.91s tid=0x00007f4260dc6800 nid=0x19a0 waiting on condition  [0x00007f41fb4f5000]
```
Типичная ситуация: приложение не завершается, потому что какой-то из потоков не закрыт и не является демоном.
Если поток анонимный, понять где именно возникла проблема очень тяжело.

Для platform threads задавайте daemon status явно. Если `daemon(boolean)` не вызван,
поток может унаследовать daemon status от создающего его потока; `ThreadFactory`
может вызываться не тем потоком, на котором был создан executor.

### Ссылки
* Concurrency In Practice (8.3.4)

## Потоки-демоны

### Мотивация
Хорошие примеры для потока-демона.
* Удаление из кеша "старых" элементов;
* Поток таймер.
Плохие примеры для потока-демона.
* Потоки, осуществляющие ввод/вывод;
* Код с finally блоком внутри потока-демона (блок может быть не выполнен, если все потоки не-демоны уже вышли).

### Ссылки
* Concurrency In Practice (7.4.2)
* [Thread: daemon and non-daemon threads](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Thread.html)

## Cached thread pool

### Мотивация
Если нет свободных потоков, готовых взять новую задачу, будет создан новый поток.
Если количество задач больше, чем может обслужить сервер в единицу времени, количество потоков будет расти и ситуация будет еще больше ухудшаться.

### Ссылки
* Effective Java, 3rd edition (Item 80).

## Fixed thread pool

### Снипет

Для JDK 21 и выше:

```java
ThreadFactory threadFactory = Thread.ofPlatform()
    .name("CoreExecutor-", 0)
    .daemon(false)
    .factory();

ExecutorService executor = new ThreadPoolExecutor(
    4,
    4,
    0L,
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(1024),
    threadFactory,
    new ThreadPoolExecutor.AbortPolicy()
);
```

### Мотивация
По-умолчанию, fixed thread pool создается с [LinkedBlockingQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/LinkedBlockingQueue.html), которая,
практически не ограничена ([MAX_VALUE](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Integer.html#MAX_VALUE)).

В `ThreadPoolExecutor` с неограниченной очередью задачи после запуска
`corePoolSize` потоков всегда попадают в очередь. Очередь не отказывает в
`offer`, поэтому дополнительные потоки до `maximumPoolSize` не создаются.
Если pool должен расширяться во время burst-нагрузки, используйте намеренно
ограниченную очередь или `SynchronousQueue` и обязательно определите rejection
policy.

Какие проблемы могут быть?
* Если ваш сервер, не успевает обслуживать запросы, они будут копиться в этой
очереди и съедать память. В конце концов, это может привести к OOM.
* Вторая, более серьезная проблема, это resource wasting. Когда thread pool
доберется до задачи в очереди, которая находится там уже давно, результат ее
вычисления может быть уже не нужен (например, клиент ушел), в этом случае,
вы зря потратите ресурсы. Такое состояние может приводить к интересной
ситуации, ваш сервер постоянно занят (перегружен), но большая часть
клиентов получает таймауты (от фронтенд, например).

Как определить оптимальный размер очереди?

На этот вопрос не легко ответить, но есть несколько советов.
### Совет 1.
Для CPU-bound задач.

```text
queue_size ≈ floor(measured_service_rate * acceptable_queue_wait)
           ≈ floor(worker_num * acceptable_queue_wait / avg_task_time)
```
*worker_num* - число одновременно работающих workers; для CPU-bound pool оно
обычно близко к доступному числу ядер

*acceptable_queue_wait* - приемлемое время ожидания задачи в очереди

*avg_task_time* - среднее время выполнения задачи

Полная оценка количества задач в системе, включая выполняющиеся задачи, равна
`worker_num + queue_size`. Поэтому множитель `1 + acceptable_queue_wait /
avg_task_time` относится к общей capacity, а не только к очереди.

Допустим, клиент готов ждать в очереди 1000 миллисекунд, а каждая задача
выполняется 5 миллисекунд, тогда размер очереди должен быть примерно равен:
`worker_num * 200`. Для максимального допустимого времени округляйте capacity
очереди вниз. Это только начальная оценка: проверьте распределения времени
выполнения, burst-нагрузку и реальную queue wait под load test.

### Совет 2.
Для I/O-bound задач.

Возможно, вам подойдет схема с фиксированным количеством потоков, которое равно количеству задач, которые вы можете обслуживать параллельно. В этом случае, можно использовать [SynchronousQueue](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/SynchronousQueue.html).

Идея здесь в том, что если вы одновременно можете сделать только N запросов (допустим 256) во внешнюю систему, то дополнительная очередь вам не нужна.

### Ссылки
* [Хорошее видео](https://www.youtube.com/watch?v=4VW4FGYHMPs&feature=youtu.be&t=296) на эту тему.

## Saturation и rejection policy

Bounded-очередь превращает бесконечное накопление задач в явную перегрузку.
Заранее определите, что увидит вызывающий код, когда одновременно заняты все
workers и заполнена очередь.

* `AbortPolicy` бросает `RejectedExecutionException`. Это хороший fail-fast
  вариант, если вызывающий код преобразует исключение в явный overload-ответ и
  rejection учитывается в метриках.
* `CallerRunsPolicy` выполняет задачу в submitter thread и может создавать
  естественный backpressure. Используйте ее только если задачу безопасно
  выполнить на вызывающем потоке. Не используйте ее для event-loop threads,
  latency-sensitive callers или когда submit выполняется под lock. После
  shutdown эта policy отбрасывает задачу. При использовании `submit` такое
  отбрасывание может оставить возвращенный `Future` незавершенным.
* `DiscardPolicy` и `DiscardOldestPolicy` допустимы только для явно lossy-задач,
  результат которых никому не нужен. Не возвращайте вызывающему коду
  незавершаемый `Future`: отменяйте и учитывайте отброшенные задачи либо
  используйте policy, которая явно сообщает о rejection.

Не считайте rejection policy универсальной: выбор зависит от API-контракта,
submitter thread и допустимости потери или синхронного выполнения задачи.

### Ссылки

* [ThreadPoolExecutor: queueing and rejected tasks](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
* [AbstractExecutorService: `submit` and `RunnableFuture`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/AbstractExecutorService.html)
* [DiscardOldestPolicy: cancel or log discarded `Future` tasks](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.DiscardOldestPolicy.html)

## Завершение executor

Завершайте только owned executor. Не закрывайте injected, shared или
framework-managed executor, если его контракт явно не передает ownership.

Перед реализацией shutdown logic найдите repository-standard lifecycle helper,
например `ExecutorShutdownAgent`. Проверьте его реализацию и существующие call
sites, затем используйте его, если контракт подходит. Не дублируйте shutdown
logic в отдельных компонентах.

Проверьте, что helper предоставляет необходимое приложению поведение:

* bounded graceful shutdown и forced-shutdown path;
* восстановление interrupt status;
* видимый результат, если lifecycle-контракт требует дождаться termination, но
  executor не завершился;
* обработку списка, возвращенного `shutdownNow()`.

Последний пункт особенно важен для задач, созданных через `submit`. Если
удаленный из очереди `Runnable` является caller-visible `Future`, переведите его
в terminal state, обычно через `cancel(false)` или `cancel(true)`, чтобы
`Future.get()` не ждал вечно. Для executor wrappers связь между возвращенным
`Runnable` и исходным `Future` может быть другой, поэтому проверьте конкретную
реализацию и call sites.

Если repository helper отсутствует, используйте стандартную двухфазную схему
JDK: `shutdown()`, bounded `awaitTermination()`, затем `shutdownNow()` и повторную
проверку termination. Для обычного shutdown `ScheduledExecutorService`
отдельный `ScheduledFuture.cancel(true)` нужен только когда требуется немедленная
отмена или interruption конкретной задачи.

Не считайте автоматическим дефектом то, что interrupt path после
`shutdownNow()` восстанавливает interrupt status и возвращается без еще одного
ожидания: такой prompt return соответствует стандартному JDK pattern. Требуйте
повторный bounded wait на этом пути только если documented lifecycle contract
гарантирует termination и может выполнить ожидание, не подавляя cancellation.

### Ссылки

* [ExecutorService shutdown and termination](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ExecutorService.html)

## Размер thread pool

### Мотивация
Как подобрать оптимальный размер thread pool для кода, где есть blocking I/O?

Ответить однозначно на этот вопрос довольно сложно, потому что есть много факторов, которые влияют на ответ. Есть некоторый разумный baseline для подобных рассуждений.

Если код использует асинхронные вычисления и не занимает поток постоянно (Completable Futures, Netty, Reactor), то большое количество тредов не нужно, а оптимальное количество не должно быть значительно больше, чем количество ядер.

Для I/O-bound задач и blocking I/O количество тредов может быть больше, чем
количество ядер, потому что не все они будут задействованы одновременно.
Формула для расчета выглядит следующим образом.
```
threads = core_num * cpu_util_coefficient * (1 + W/C)
```
*core_num* - количество ядер

*cpu_util_coefficient* - желаемый процент утилизации cpu (в процентах, отнормированных на 1)

*W/C* - соотношение времени ожидания (I/O, ожидание захвата монитора и статусы потоков WAITING/TIMED_WAITING) к времени вычислений (CPU)

Измеряйте `W` и `C` внутри задачи после начала ее выполнения. Не включайте
ожидание в очереди самого executor: это следствие текущей конфигурации, а не
свойство workload. Результат формулы округляйте вверх, затем ограничивайте
реальной capacity downstream-систем и проверяйте load test.

Если значительную часть `W` составляет lock contention, сначала исследуйте
критические секции: добавление platform threads может только увеличить это
ожидание.

### Пример 1.
У нас есть микро-сервис, который отвечает за 100 миллисекунд, при этом сами вычисления занимают 5 миллисекунд (мы поняли это по профайлеру, например). Всего 16 ядер. И мы хотим, чтобы утилизация cpu была не выше 30%.
```
W = 100 - 5 = 95 миллисекунд
16 * 0.3 * (1 + 95/5) = 96 тредов.
```
Так много тредов получается, потому что у нас время ожидания много больше времени вычисления.
### Пример 2.
У нас есть микро-сервис, который отвечает за 5 миллисекунд (чтение только из памяти), при этом сами вычисления занимают 3 миллисекунды. Всего 16 ядер. И мы хотим, чтобы утилизация cpu была не выше 40%.
```
W = 5 - 3 = 2 миллисекунды
16 * 0.4 * (1 + 2/3) ≈ 10.67, округляем вверх до 11 тредов.
```
Обратите внимание, чем меньше соотношение I/O-bound / CPU-bound, тем меньше нужно тредов.

### Ссылки
* Concurrency In Practice (8.2)

## Virtual threads

Для JDK 21 и выше можно использовать виртуальные потоки (virtual threads) для blocking I/O и писать простой синхронный код. Они повышают throughput приложения, но не уменьшают latency отдельной задачи. Virtual threads подходят для большого количества задач, которые проводят основное время в ожидании, но не для длительных CPU-bound вычислений.

Примеры использования virtual threads в коде:

* Запросы в базу данных
* Вызовы другого сервиса по сети
* Модель приложения thread per request

### Создание virtual threads

Не объединяйте virtual threads в pool и не применяйте к ним формулу `W/C`.
Создавайте отдельный virtual thread для каждой concurrent-задачи. Executor в
этом примере является lifecycle и submission abstraction, а не ограниченным
пулом workers:

```java
ThreadFactory threadFactory = Thread.ofVirtual()
        .name("Request-", 0)
        .factory();

ExecutorService executor = Executors.newThreadPerTaskExecutor(threadFactory);
```

Создавайте такой executor на уровне компонента или приложения, а не для каждой
задачи, и закрывайте его вместе с lifecycle владельца. Если имена не нужны,
используйте `Executors.newVirtualThreadPerTaskExecutor()`.

Имя virtual thread полезно для диагностики, но его отсутствие само по себе не
является дефектом.

Перед добавлением отдельного `Semaphore` или bulkhead проверьте уже существующие
ограничители: connection pool, rate limiter, bounded request queue или upstream
admission control. Добавляйте еще один limiter только если нужно отдельно
ограничить число ожидающих задач, fail-fast поведение или долю конкретного
компонента. Не заменяйте такой limiter пулом virtual threads.

Не используйте `ThreadLocal` для кеширования дорогих объектов при переходе на
virtual threads: число экземпляров может вырасти вместе с числом
concurrent-задач. Контекст запроса в `ThreadLocal` допустим, если его lifecycle
и очистка явно определены; подробнее смотрите guidance про task lifecycle and
context из основного skill.

### Совместимость JDK

Подводные камни:

| Версия JDK | Проблема                                                                                                             |
|:-----------|:---------------------------------------------------------------------------------------------------------------------|
| JDK 21-23  | Блокировка внутри `synchronized`, ожидание монитора и выполнение native/FFM-кода могут пинить carrier thread           |
| JDK 24+    | `synchronized` больше не пинит carrier thread; выполнение native-метода или foreign function все еще может это сделать |

### Ссылки

* [Oracle: Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
* [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
* [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
