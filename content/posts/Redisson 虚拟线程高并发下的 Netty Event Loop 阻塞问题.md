+++
title = "Redisson 虚拟线程高并发下的 Netty Event Loop 阻塞问题"
date = "2026-07-26"

[taxonomies]
tags=["Redis", "问题排查", "高并发"]

[extra]
comment = true
+++

Redisson Issue #7155 描述了一个典型的高并发问题：应用使用 Java 虚拟线程后，在运行一段时间后偶发出现 `Command still hasn't been written into connection!`。表面看，这是 Redis 命令写入超时；深入线程栈后可以看到，真正的问题发生在 Redisson 连接释放路径上：Netty event loop 线程在释放连接时同步进入连接池等待队列，并卡在一个 `ReentrantLock` 上。

这篇文章会从问题现象开始，逐步解释 Redisson 连接池、`AsyncSemaphore`、`FastRemovalQueue`、Netty event loop 的关系，最后说明修复方案、测试方式和 trade-off。

## 问题背景

issue 中的用户场景是：

- 使用 Java virtual threads；
- 高并发访问 Redis；
- Redisson 运行一段时间后报错；
- 同样业务换成普通线程池后问题不容易出现。

报错信息的关键部分是：

```text
org.redisson.client.RedisTimeoutException: Command still hasn't been written into connection!
Netty pending tasks: 1704
command: (GET)
```

这不是 Redis 返回慢，也不是 Redis 命令执行失败，而是 Redisson 已经准备发送命令，但命令没有及时写到连接里。

更关键的是用户提供的线程栈：

```text
redisson-netty-8-5
  at java.util.concurrent.locks.ReentrantLock.lock
  at org.redisson.misc.WrappedLock.execute
  at org.redisson.misc.FastRemovalQueue$DoublyLinkedList.removeFirst
  at org.redisson.misc.FastRemovalQueue.poll
  at org.redisson.misc.AsyncSemaphore.tryRun
  at org.redisson.misc.AsyncSemaphore.tryForkAndRun
  at org.redisson.misc.AsyncSemaphore.release
  at org.redisson.connection.ConnectionsHolder.releaseConnection
  at org.redisson.connection.ClientConnectionsEntry.returnConnection
  at org.redisson.connection.MasterSlaveEntry.releaseWrite
  at org.redisson.command.RedisExecutor.release
```

这里最重要的信号是：`redisson-netty-*` 线程卡在 `ReentrantLock.lock()`。

Netty event loop 是 Redisson 网络 IO 的核心线程。它负责处理 socket 读写、channel 回调和网络事件。如果它被普通业务锁阻塞，后续 Redis 命令就可能排队但写不出去，最终形成 `Command still hasn't been written into connection!`。

## Redisson 连接池的大致工作方式

Redisson 不会每条 Redis 命令都新建 TCP 连接。它维护连接池：

```text
业务请求
  -> 申请连接池 permit
  -> 拿空闲 RedisConnection 或创建新连接
  -> 发送 Redis 命令
  -> 命令完成后归还连接
  -> 释放连接池 permit
```

在这个流程里，连接池使用 `AsyncSemaphore` 控制并发连接数。

相关类是：

- `ConnectionsHolder`：管理某个 Redis 节点的连接集合；
- `AsyncSemaphore`：异步信号量，用于控制连接池最大并发；
- `FastRemovalQueue`：保存等待 permit 的 future 队列；
- `RedisExecutor`：负责命令执行、连接获取和释放。

## AsyncSemaphore 的角色

`AsyncSemaphore` 可以理解成异步版信号量。

传统信号量可能是：

```java
semaphore.acquire(); // 没 permit 时阻塞当前线程
```

Redisson 这里不能直接阻塞线程，所以它返回一个 future：

```java
CompletableFuture<Void> f = semaphore.acquire();
```

如果有 permit，future 会完成；如果没有 permit，future 会进入等待队列。等别人调用 `release()` 时，再完成等待队列里的某个 future。

核心字段如下：

```java
private final ExecutorService executorService;
private final AtomicInteger tasksLatch = new AtomicInteger(1);
private final AtomicInteger stackSize = new AtomicInteger();

private final AtomicInteger counter;
private final FastRemovalQueue<CompletableFuture<Void>> listeners = new FastRemovalQueue<>();
```

含义分别是：

- `counter`：当前可用 permit 数；
- `listeners`：等待 permit 的 future 队列；
- `executorService`：可选 executor，用于把部分唤醒工作放到其他线程；
- `stackSize` / `tasksLatch`：原有逻辑中用于避免 future 回调递归过深。

## 原始 release 逻辑的问题

修复前的 `release()` 逻辑很短：

```java
public void release() {
    counter.incrementAndGet();
    tryForkAndRun();
}
```

`tryForkAndRun()` 的逻辑是：如果当前 future completion 嵌套太深，就把 `tryRun()` 提交到 executor；否则直接执行 `tryRun()`。

```java
private void tryForkAndRun() {
    if (executorService != null) {
        int val = tasksLatch.get();
        if (stackSize.get() > 25 * val
                && tasksLatch.compareAndSet(val, val+1)) {
            executorService.submit(() -> {
                tasksLatch.decrementAndGet();
                tryRun();
            });
            return;
        }
    }

    tryRun();
}
```

这段代码原本有合理用意：避免 future 回调链嵌套过深。但是它不能解决当前 issue，因为大多数情况下它仍然会同步执行：

```java
tryRun();
```

如果 `release()` 的调用线程是 Netty event loop，那么 Netty event loop 就会同步进入 `tryRun()`。

而 `tryRun()` 会调用：

```java
CompletableFuture<Void> future = listeners.poll();
```

这正是线程栈里的阻塞点。

## listeners.poll() 为什么会阻塞

`listeners` 的类型是：

```java
private final FastRemovalQueue<CompletableFuture<Void>> listeners = new FastRemovalQueue<>();
```

`FastRemovalQueue.poll()` 的代码是：

```java
public E poll() {
    Node<E> node = list.removeFirst();
    if (node != null) {
        index.remove(node.value);
        return node.value;
    }
    return null;
}
```

这里调用了 `list.removeFirst()`。

`DoublyLinkedList` 内部有锁：

```java
static class DoublyLinkedList<E> implements Iterable<E> {
    private final WrappedLock lock = new WrappedLock();
```

`removeFirst()` 会通过 `lock.execute(...)` 修改链表头尾指针：

```java
public Node<E> removeFirst() {
    return lock.execute(() -> {
        Node<E> currentHead = head;
        if (head == tail) {
            head = null;
            tail = null;
        } else {
            head = head.next;
            head.prev = null;
        }
        if (currentHead != null) {
            currentHead.setDeleted();
        }
        return currentHead;
    });
}
```

`WrappedLock` 本质上是一个 `ReentrantLock`：

```java
private final Lock lock = new ReentrantLock();

public <T> T execute(Supplier<T> r) {
    lock.lock();
    try {
        return r.get();
    } finally {
        lock.unlock();
    }
}
```

因此调用链是：

```text
listeners.poll()
  -> FastRemovalQueue.poll()
  -> DoublyLinkedList.removeFirst()
  -> WrappedLock.execute()
  -> ReentrantLock.lock()
```

如果另一个线程正在持有这把锁，比如正在 `add`、`remove` 或 `poll`，当前线程调用 `lock.lock()` 时就会被 park，等待锁释放。

普通线程等待这把锁通常只是延迟。但 Netty event loop 等待这把锁，就可能让网络写任务堆积。

## 为什么虚拟线程更容易暴露问题

虚拟线程不是问题本身。问题本身是 Netty event loop 不应该同步竞争连接池等待队列的锁。

但虚拟线程会放大并发程度。相比固定大小线程池，虚拟线程场景可能同时产生更多 Redis 调用。这样会造成：

```text
更多 acquire -> listeners.add()
更多 timeout/cancel -> listeners.remove()
更多 release -> listeners.poll()
```

这些操作都围绕 `FastRemovalQueue` 的链表锁。并发越高，Netty event loop 在 release 路径上撞到锁竞争的概率越大。

这解释了 issue 里的现象：普通线程池不容易出现，虚拟线程高并发更容易出现。

## getGroup 和 getExecutor 的区别

原来的 `ConnectionsHolder` 构造方法使用：

```java
this.freeConnectionsCounter = new AsyncSemaphore(poolMaxSize, serviceManager.getGroup());
```

`serviceManager.getGroup()` 返回 Netty `EventLoopGroup`。这是网络 IO 线程组，负责 Redis 连接的读写。

修复后改成：

```java
this.freeConnectionsCounter = new AsyncSemaphore(poolMaxSize, serviceManager.getExecutor());
```

`serviceManager.getExecutor()` 是 Redisson 的普通 worker executor。它更适合执行非网络 IO 的内部调度任务。

可以简单区分：

```text
getGroup()    -> Netty 网络线程，不能阻塞
getExecutor() -> Redisson worker 线程，可以做普通后台任务
```

为什么 `getGroup()` 更危险？

不是因为它更容易拿不到锁，而是因为它一旦卡住，后果更严重。

如果 worker executor 卡住，影响的是普通后台任务或连接等待者唤醒速度。如果 Netty event loop 卡住，影响的是 Redis 命令写入、响应读取和网络事件处理。issue 中的 `Netty pending tasks: 1704` 就说明 Netty 任务已经堆积。

## 修复方案

修复包含两部分。

### 1. 修改 AsyncSemaphore.release()

修复后的代码：

```java
public void release() {
    counter.incrementAndGet();
    if (listeners.isEmpty()) {
        return;
    }
    if (executorService != null) {
        try {
            executorService.execute(this::tryRun);
            return;
        } catch (RejectedExecutionException e) {
            // fallback to the caller thread during shutdown
        }
    }
    tryRun();
}
```

这段代码做了几件事：

1. `counter.incrementAndGet()` 仍然先归还 permit；
2. 如果没有等待者，直接返回，避免无意义进入队列锁；
3. 如果有 executor，把唤醒等待者的 `tryRun()` 提交到 executor；
4. 如果 executor 拒绝任务，比如 shutdown 期间，就 fallback 到当前线程执行 `tryRun()`，避免 permit 丢失。

这和原来的 `tryForkAndRun()` 不同。`tryForkAndRun()` 是“递归过深才异步”，而这里需要的是“release 有 executor 时尽量不要同步执行”。这是为保护 Netty event loop 做出的明确调整。

### 2. 修改 ConnectionsHolder 的 executor 选择

原来：

```java
this.freeConnectionsCounter = new AsyncSemaphore(poolMaxSize, serviceManager.getGroup());
```

修复后：

```java
this.freeConnectionsCounter = new AsyncSemaphore(poolMaxSize, serviceManager.getExecutor());
```

这使连接池等待者唤醒不再使用 Netty event loop，而是使用 Redisson worker executor。

## 回归测试设计

### AsyncSemaphoreTest

新增测试 `testReleaseWithExecutorCompletesWaiterAsynchronously`：

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
CountDownLatch taskStarted = new CountDownLatch(1);
CountDownLatch unblock = new CountDownLatch(1);

executor.execute(() -> {
    taskStarted.countDown();
    unblock.await();
});
assertThat(taskStarted.await(1, TimeUnit.SECONDS)).isTrue();

AsyncSemaphore semaphore = new AsyncSemaphore(0, executor);

CompletableFuture<Void> waiter = semaphore.acquire();
semaphore.release();

assertThat(waiter).isNotDone();

unblock.countDown();
waiter.get(1, TimeUnit.SECONDS);

assertThat(waiter).isCompleted();
```

这个测试证明：有 executor 时，`release()` 不会同步完成等待者，而是提交任务。

新增测试 `testReleaseFallsBackIfExecutorRejectsWakeup`：

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
executor.shutdown();
AsyncSemaphore semaphore = new AsyncSemaphore(0, executor);

CompletableFuture<Void> waiter = semaphore.acquire();

assertThatCode(semaphore::release).doesNotThrowAnyException();
assertThat(waiter).isCompleted();
assertThat(semaphore.getCounter()).isZero();
```

这个测试覆盖 executor shutdown 或拒绝任务的 fallback 场景，确保不抛异常、不丢 permit。

### ConnectionsHolderTest

新增测试 `testConnectionCounterUsesServiceExecutorToWakeWaiter`：

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(1, 1, 0, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<>());
CountDownLatch taskStarted = new CountDownLatch(1);
CountDownLatch unblock = new CountDownLatch(1);
MasterSlaveConnectionManager manager = buildManager(executor);

executor.execute(() -> {
    taskStarted.countDown();
    unblock.await();
});
Assertions.assertThat(taskStarted.await(1, TimeUnit.SECONDS)).isTrue();

ConnectionsHolder<RedisConnection> holder =
        new ConnectionsHolder<>(null, 1, r -> new CompletableFuture<>(), manager.getServiceManager(), false);
AsyncSemaphore counter = holder.getFreeConnectionsCounter();

CompletableFuture<Void> acquired = counter.acquire();
Assertions.assertThat(acquired).isCompleted();

CompletableFuture<Void> waiter = counter.acquire();
Assertions.assertThat(waiter).isNotDone();

counter.release();

Assertions.assertThat(waiter).isNotDone();
Assertions.assertThat(executor.getQueue()).hasSize(1);

unblock.countDown();

waiter.get(1, TimeUnit.SECONDS);
Assertions.assertThat(waiter).isCompleted();
```

这个测试保护的是连接池集成路径。它证明 `ConnectionsHolder` 的 semaphore 使用的是 `serviceManager.getExecutor()`，而不是 Netty group。

## 测试结果

运行：

```bash
mvn -pl redisson -Punit-test -DskipITs -Dtest=org.redisson.misc.AsyncSemaphoreTest#testReleaseWithExecutorCompletesWaiterAsynchronously,org.redisson.connection.ConnectionsHolderTest#testConnectionCounterUsesServiceExecutorToWakeWaiter test
```

结果：

```text
Tests run: 2, Failures: 2, Errors: 0, Skipped: 0
```

恢复修复后运行完整相关测试：

```bash
mvn -pl redisson -Punit-test -DskipITs -Dtest=org.redisson.misc.AsyncSemaphoreTest,org.redisson.connection.ConnectionsHolderTest test
```

结果：

```text
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

这说明测试具备回归价值：修复前失败，修复后通过。

## Trade-off 与风险

### Trade-off 1：多了一次 executor 调度

原来 `release()` 大多数时候直接同步执行 `tryRun()`。修复后，只要存在等待者且配置了 executor，就会提交 executor task。

这会增加一次调度成本，但换来的是 Netty event loop 不再同步抢等待队列锁。对于网络 IO 线程来说，这是值得的。

### Trade-off 2：service executor 变得更重要

`ConnectionsHolder` 现在使用 `serviceManager.getExecutor()` 唤醒连接等待者。

如果用户配置了很小、很慢或经常阻塞的 executor，连接等待者唤醒可能变慢。但这比阻塞 Netty event loop 更可控。

### Trade-off 3：没有重构 FastRemovalQueue

保留 `FastRemovalQueue` 意味着队列内部仍然有锁。修复不是消除锁，而是避免 Netty event loop 直接同步等待这把锁。

这是一个更小范围、更安全的修复。

## 结论

Issue #7155 的根因不是简单的 Redis 超时，也不是虚拟线程本身有问题，而是虚拟线程高并发放大了 Redisson 连接池等待队列的锁竞争，导致 Netty event loop 在连接释放路径上可能被阻塞。

修复的关键原则是：

```text
Netty event loop 不应该执行可能阻塞的连接池等待队列唤醒逻辑。
```

具体落地为：

- `AsyncSemaphore.release()` 在有 executor 时异步执行 `tryRun()`；
- `ConnectionsHolder` 使用 Redisson worker executor，而不是 Netty event loop group；
- 增加回归测试，证明修复前失败、修复后通过。

这个修复没有扩大 API，没有引入依赖，也没有重写共享队列结构，属于针对根因的局部修复。

## 相关文件

- `redisson/src/main/java/org/redisson/misc/AsyncSemaphore.java`
- `redisson/src/main/java/org/redisson/connection/ConnectionsHolder.java`
- `redisson/src/main/java/org/redisson/misc/FastRemovalQueue.java`
- `redisson/src/main/java/org/redisson/misc/WrappedLock.java`
- `redisson/src/test/java/org/redisson/misc/AsyncSemaphoreTest.java`
- `redisson/src/test/java/org/redisson/connection/ConnectionsHolderTest.java`
