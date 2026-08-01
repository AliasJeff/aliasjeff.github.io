+++
title = "Redisson Pub/Sub 订阅失败后的 Semaphore Permit 泄漏问题"
date = "2026-08-01"

[taxonomies]
tags=["Redis", "Redisson", "问题排查", "高并发"]

[extra]
comment = true
+++

Redisson Issue #7264 描述的是一个 Pub/Sub 订阅锁泄漏问题：某次订阅失败或超时后，`PublishSubscribeService.locks[]` 中某个 channel stripe 的 `AsyncSemaphore` permit 可能永久丢失。之后 hash 到同一个 stripe 的 channel，再做订阅或取消订阅时会一直拿不到锁，最终固定在 `subscriptionTimeout` 后失败。

这篇文章会从 issue 的现象开始，解释 Redisson Pub/Sub 的 channel stripe semaphore 是怎么用的，再展开这次修复中能确定复现的一条路径：`subscribeNoTimeout(...)` 在 `MasterSlaveEntry == null` 时提前返回，但没有释放外层已经拿到的 semaphore。

需要先说明边界：这个修复不是覆盖 issue 里提到的所有风险路径，而是修复其中一条明确、可复现、能用单元测试证明的 permit 泄漏路径。

## 问题背景

issue 中的核心现象是：

- Redis 节点整体可用；
- 大部分 Pub/Sub channel 仍然正常；
- 一小部分 channel 的订阅或取消订阅持续超时；
- 超时信息集中在获取 subscription lock 这一步；
- JVM 重启后恢复。

报错类似：

```text
org.redisson.client.RedisTimeoutException:
Unable to acquire subscription lock after <subscriptionTimeout>ms.
Try to increase 'subscriptionTimeout', 'subscriptionsPerConnection',
'subscriptionConnectionPoolSize' parameters.
```

从报错看，很容易先怀疑是 `subscriptionTimeout` 太小、订阅连接池不够、Redis 变慢，或者网络抖动。但 issue 的关键点不在 Redis 命令执行慢，而在客户端内部的订阅锁拿不到。

如果只是 Redis 短暂不可用，后续重试或下一次订阅应该还有机会恢复。现在的问题是部分 channel 会持续失败，直到进程重启。这更像是客户端内部某个 permit、锁或状态被卡住了。

## Channel Stripe Semaphore 是什么

Redisson Pub/Sub 里不是每个 channel 都有一个独立锁，而是使用一组 `AsyncSemaphore` 做分片。代码里可以看到：

```java
AsyncSemaphore getSemaphore(ChannelName channelName) {
    return locks[Math.abs(channelName.hashCode() % locks.length)];
}
```

这意味着：

```text
channelName
  -> hashCode
  -> 映射到 locks[] 的某个下标
  -> 使用该下标上的 AsyncSemaphore 串行化订阅相关操作
```

这种做法的目的很直接：避免同一个 channel 或同一组映射到相同 stripe 的 channel 在订阅、取消订阅、entry 复用等流程中并发修改共享状态。

所以这个 semaphore 不是普通临时对象。它是 Pub/Sub 服务内部长期存在的 stripe 锁。某一次失败路径如果拿了 permit 但没有归还，会影响之后所有落到同一 stripe 的操作。

## AsyncSemaphore 在这里怎么工作

`AsyncSemaphore` 可以理解成异步版信号量：

```java
CompletableFuture<Void> future = semaphore.acquire();
```

有 permit 时，future 立即完成；没有 permit 时，future 会一直 pending，等其他路径调用：

```java
semaphore.release();
```

在 Pub/Sub 订阅链路里，外层先拿 stripe semaphore，然后进入真正的订阅逻辑：

```text
PublishSubscribe.subscribe(...)
  -> service.getSemaphore(new ChannelName(channelName))
  -> semaphore.acquire()
  -> service.subscribeNoTimeout(..., semaphore, listener)
```

关键点是：`subscribeNoTimeout(...)` 收到的 `semaphore` 已经是外层拿到的 permit。进入这个方法后，如果流程提前失败，它必须负责把 permit 还回去。否则外层没有别的地方能替它释放。

## 问题调用链

锁订阅路径在 `PublishSubscribe.subscribe(...)` 中：

```java
public CompletableFuture<E> subscribe(String entryName, String channelName, int permits) {
    AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
    CompletableFuture<E> newPromise = new CompletableFuture<>();

    semaphore.acquire().thenAccept(c -> {
        ...
        CompletableFuture<PubSubConnectionEntry> s =
                service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, listener);
        ...
    });

    return newPromise;
}
```

这个方法先根据 channel 找到 stripe semaphore，然后 acquire。拿到 permit 后，它会创建 entry，再调用：

```java
service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, listener);
```

注意这里没有在调用后立刻释放 semaphore。原因是订阅流程还没结束，后面的异步逻辑要继续持有这把锁，直到订阅成功、失败或超时路径明确处理完成。

问题发生在 `PublishSubscribeService.subscribeNoTimeout(...)` 的早退分支：

```java
CompletableFuture<PubSubConnectionEntry> subscribeNoTimeout(Codec codec, String channelName,
                                                            AsyncSemaphore semaphore,
                                                            RedisPubSubListener<?>... listeners) {
    MasterSlaveEntry entry = getEntry(new ChannelName(channelName));
    if (entry == null) {
        int slot = connectionManager.calcSlot(channelName);
        return connectionManager.getServiceManager().createNodeNotFoundFuture(channelName, slot);
    }

    ...
}
```

旧逻辑在 `entry == null` 时直接返回一个 node-not-found failed future。这个 future 表达了“订阅失败”，但它没有处理“前面拿到的 semaphore 要归还”。

完整问题链路是：

```text
PublishSubscribe.subscribe(entryName, channelName)
  -> 根据 channelName 找到 stripe semaphore
  -> semaphore.acquire() 成功，permit 从 1 变成 0
  -> 调用 subscribeNoTimeout(...)
  -> getEntry(new ChannelName(channelName)) 返回 null
  -> 旧代码直接返回 createNodeNotFoundFuture(...)
  -> semaphore.release() 没有执行
  -> 这个 stripe 的 permit 永久停在 0
  -> 后续相同 stripe 的订阅/取消订阅一直 pending
  -> 等到 subscriptionTimeout 后报 Unable to acquire subscription lock
```

这就是 issue 中“一部分 channel 固定超时、重启恢复”的合理解释。不是每个 channel 都失败，因为 semaphore 是按 stripe 分片的；只有落到同一把 semaphore 的 channel 会被影响。

## 这是猜测还是确定的问题

对这条 `entry == null` 路径来说，permit 泄漏不是猜测，而是代码行为直接推出的结果。

旧代码满足三个条件：

1. 外层已经调用 `semaphore.acquire()`；
2. `subscribeNoTimeout(...)` 收到了这个已经被占用的 semaphore；
3. `entry == null` 分支直接 `return`，没有 `semaphore.release()`。

在 `AsyncSemaphore(1)` 的情况下，第一次 acquire 后可用 permit 变成 0。如果没有 release，第二次 acquire 返回的 future 就不会完成。这一点可以不依赖真实 Redis，用单元测试确定性复现。

不过也要区分两件事：

- 这条路径本身会泄漏 permit，是确定的；
- 线上 issue 的所有现象是否全部由这条路径造成，仅靠当前单元测试不能证明。

所以这次 PR 的描述应该克制：修复 `entry == null` 这个直接泄漏路径，而不是宣称已经覆盖 #7264 中所有可能路径。

## 为什么 entry 会是 null

`getEntry(new ChannelName(channelName))` 最终依赖当前连接管理器里的 Redis 节点映射。集群拓扑变化、slot 覆盖暂时不可用、节点发现还没完成，或者连接管理器还没有对应写 entry 时，都可能让这里拿不到 `MasterSlaveEntry`。

Redisson 原本也知道这种情况可能发生，所以旧代码没有继续订阅，而是返回：

```java
connectionManager.getServiceManager().createNodeNotFoundFuture(channelName, slot)
```

这个返回值本身没问题。问题是这个失败发生在外层已经拿到 stripe semaphore 之后。失败可以直接返回，但资源也要一起释放。

## 修复方案

修复很小，只在 `entry == null` 分支返回前归还 semaphore：

```java
CompletableFuture<PubSubConnectionEntry> subscribeNoTimeout(Codec codec, String channelName,
                                                            AsyncSemaphore semaphore,
                                                            RedisPubSubListener<?>... listeners) {
    MasterSlaveEntry entry = getEntry(new ChannelName(channelName));
    if (entry == null) {
        semaphore.release();
        int slot = connectionManager.calcSlot(channelName);
        return connectionManager.getServiceManager().createNodeNotFoundFuture(channelName, slot);
    }

    ...
}
```

这个位置释放是合理的，因为 `entry == null` 时订阅流程没有进入后续异步链路，也没有创建真实的 pub/sub connection entry。当前方法已经决定提前失败返回，继续持有 stripe semaphore 没有意义。

修复后的流程变成：

```text
entry == null
  -> release stripe semaphore
  -> 返回 node-not-found failed future
```

也就是说，业务结果仍然是失败，异常语义没有变；只是失败路径不再把内部锁泄漏掉。

## 单元测试

新增测试是：

```text
PublishSubscribeTest.testSubscribeNoTimeoutReleasesSemaphoreIfEntryIsMissing
```

测试没有启动 Redis，也没有搭 Redis Cluster。它做的是更小的确定性复现：直接模拟 `getEntry(...) == null`。

核心步骤是：

```java
AsyncSemaphore semaphore = new AsyncSemaphore(1);

// Simulate the caller already holding the semaphore permit.
semaphore.acquire().join();
CompletableFuture<Void> waiter = semaphore.acquire();
assertFalse(waiter.isDone());

CompletableFuture<PubSubConnectionEntry> subscribeFuture =
        service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, new BaseRedisPubSubListener());

assertSame(failedFuture, subscribeFuture);
assertTrue(subscribeFuture.isCompletedExceptionally());
assertTrue(waiter.isDone());
```

这里先手动 acquire 一次 semaphore，是为了模拟外层 `PublishSubscribe.subscribe(...)` 已经持有 stripe permit 的真实调用状态。然后再发起第二次 acquire，得到一个 pending 的 waiter。

如果 `subscribeNoTimeout(...)` 在失败路径没有 release，`waiter` 会一直 pending。修复后，`entry == null` 分支会 release，`waiter` 立即完成。

测试：

```text
订阅失败是允许的；
node-not-found future 也是允许的；
但失败路径不能吞掉 channel stripe semaphore permit。
```

## 为什么这是有效复现

这个测试没有复现线上所有条件，比如 Redis Cluster 拓扑变化、真实节点下线、高并发订阅等。但它真实还原了当前修复对应的最小场景：

```text
外层已经持有 semaphore
  -> subscribeNoTimeout(...) 查不到 MasterSlaveEntry
  -> 方法提前返回 failed future
  -> 验证 semaphore 是否归还
```

这比集成测试更适合保护这条早退路径。集成测试可能更接近生产环境，但要稳定制造 `getEntry(...) == null` 很难，而且容易被网络、Redis 启动速度、slot 分配等因素影响。这里的 bug 本质是一个资源释放遗漏，用单元测试直接卡住这个分支更准确。

如果 reviewer 追问 “这个测试是不是完整复现 #7264”，答案应该是：

```text
No. It reproduces one direct leak path described in #7264:
subscribeNoTimeout(...) returns early when no MasterSlaveEntry is available,
after the caller has already acquired the channel stripe semaphore.
```

精确复现一条子路径，而不是完整复现所有可能路径

目标测试：

```bash
mvn -pl redisson -Punit-test -DskipITs -Dtest=org.redisson.pubsub.PublishSubscribeTest test
```

格式和项目检查：

```bash
mvn -pl redisson -Punit-test -DskipITs checkstyle:check
mvn -pl redisson -Punit-test -DskipITs license:check
```

## 结论

Issue #7264 的核心风险是 Pub/Sub channel stripe semaphore 的 permit 泄漏。当前修复确认并解决了一条明确路径：外层已经拿到 semaphore 后，`subscribeNoTimeout(...)` 因为找不到 `MasterSlaveEntry` 提前失败返回，但旧代码没有 release。

修复只是在早退前补上 `semaphore.release()`。业务仍然返回 node-not-found failed future，但内部 stripe semaphore 不会被永久占住。对应单元测试证明了旧逻辑会让后续 acquire 卡住，而修复后 waiter 能立即完成。

## 相关文件

- `redisson/src/main/java/org/redisson/pubsub/PublishSubscribe.java`
- `redisson/src/main/java/org/redisson/pubsub/PublishSubscribeService.java`
- `redisson/src/test/java/org/redisson/pubsub/PublishSubscribeTest.java`
- `redisson/src/main/java/org/redisson/misc/AsyncSemaphore.java`
