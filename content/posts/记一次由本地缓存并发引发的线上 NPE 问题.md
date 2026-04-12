+++
title = "记一次由本地缓存并发引发的线上 NPE 问题"
date = "2026-04-14"

[taxonomies]
tags=["高并发", "问题排查", "SpringBoot"]

[extra]
comment = true
+++


## 一、本地缓存代码

简单来说，这段业务代码的作用是做了一个本地缓存。系统里有一张配置表，业务在读取配置时，为了减轻数据库查询压力并提升性能，希望能尽量走本地缓存。由于我们的业务场景对配置数据一致性的要求没有那么苛刻（集群环境下，节点间存在短暂的数据不一致是可以接受的），所以直接使用了 AI 写的一个极其简单的单机本地缓存方案。

以下是出问题的原始代码：

```
@Component
public class ConfigCache {

    // 使用了 static 和 final 修饰的 HashMap 作为本地缓存
    private static final Map<String, String> CACHE = new HashMap<>();

    @Autowired
    private ConfigDAO configDAO;

    // Spring 容器启动后，初始化加载缓存
    @PostConstruct
    public void init() {
        load();
    }

    // 定时任务：每 5 分钟重新刷新一次本地缓存
    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void reload() {
        load();
    }

    // 上层业务调用的获取配置方法
    public String get(String key) {
        return CACHE.get(key);
    }

    // 核心加载逻辑
    private void load() {
        List<ConfigDO> configs = configDAO.listAll();
        CACHE.clear(); // 问题点：先把缓存清空
        for (ConfigDO config : configs) {
            CACHE.put(config.getKey(), config.getValue()); // 然后再循环 put 进去
        }
    }
}
```

这段代码发布上线后，系统**每隔几天就会出现一次 `NullPointerException`（空指针异常）**，报错提示某一个配置没有拿到。而且诡异的是，每隔几天报错时，拿不到的配置 Key 都不一样，完全是随机的。

------

## 二、并发竞态问题

在这个场景中，存在两类线程在同时访问 `CACHE`：

1. **写线程**：后台的定时任务线程，每 5 分钟执行一次 `reload` 刷新。
2. **读线程**：前端大量的业务工作线程，高频调用 `get` 方法。

问题就出在 `load()` 方法里：当定时任务线程执行了 `CACHE.clear()` 清空缓存，**但还没有来得及把新数据全部 `put` 进去的这段微小时间窗口内**，如果恰好有一个业务线程来调用 `get(key)`，它面对的必然是一个空空如也的 Map。

结果显而易见：业务线程必定拿到一个 `null` 值。由于这个时间窗口很小，所以偶尔才会触发，并且每次触发时请求的 Key 都不固定。

------

## 三、解决方案

### 方案 1：简单粗暴地加锁（不可取）

简单在方法上加上 synchronized 互斥锁，这样确实能解决并发问题，但**性能极差**。我们既然用上了缓存，说明这里的请求量应该不小。如果在刷新缓存（查数据库 + 全量循环写入）的期间，所有的 `get` 读请求全都要被阻塞在这里等待，这绝对是一种极其低效的操作。

### 方案 2：以空间换时间（推荐方案）

我们在 `load` 时，不要去清理原有的 Map，而是**直接开辟一个新空间，实例化一个新的 Map**，把查出来的数据 `put` 到新 Map 里。等数据全都装载完毕之后，再把 `CACHE` 的引用瞬间指向这个新的 Map。

这种思想在业界应用非常广泛。比如 **MySQL 的 Online DDL** 操作，大表增加字段直接改会锁表，它的底层解决方案就是重新开辟一块空间，把数据复制过去，数据复制完毕后只在切换新老表的瞬间加一段极短的锁。

代码初步改造如下（注意：去掉了 `final` 关键字）：

```
// 去掉 final，因为我们需要改变引用指向
private static Map<String, String> CACHE = new HashMap<>();

private void load() {
    List<ConfigDO> configs = configDAO.listAll();
    
    // 1. 新起一个局部 Map（开辟新空间）
    Map<String, String> newCache = new HashMap<>(); 
    
    // 2. 将数据装载到新 Map 中
    for (ConfigDO config : configs) {
        newCache.put(config.getKey(), config.getValue());
    }
    
    // 3. 瞬间切换引用（引用的赋值在 JVM 中是原子操作）
    CACHE = newCache; 
}
```

------

## 四、为什么要加 `volatile` 关键字？

定时任务是一个独立线程，业务 `get` 请求是另外的线程。在 Java 内存模型（JMM）中，**线程之间的可见性生效时间是不可控的**。因此，必须给 `CACHE` 加上 `volatile` 关键字：

```
private static volatile Map<String, String> CACHE = new HashMap<>();
```

不过，如果仅仅是为了保证可见性，在集群部署的情况下，各台机器刷新缓存的时间本来就很难保证完全一致（集群视角下一会儿读到老值，一会儿读到新值），所以单机用 `volatile` 去保证可见性并不是我们的主要矛盾。

**但是，这里必须加 `volatile` 的核心目的，是为了防止指令重排序**

在 `CACHE = newCache;` 这一步，在底层其实包含了内存分配、对象初始化和引用赋值等多步操作。如果没有 `volatile` 建立的内存屏障，编译器和 CPU 极有可能发生指令重排序，执行顺序可能变成：

1. 分配一个 Map 的内存空间。
2. **提前把 `CACHE` 变量的引用指向这块内存。**
3. **再去执行循环 `put` 的装载操作。**

如果发生了重排序，`CACHE` 已经指向了新的内存地址，但这个新对象**还没来得及把所有的值 put 进去**。此时，如果一个高并发的业务 `get` 请求进来了，获取到了最新的引用，但去里面拿数据时，依然会拿到一个处于半初始化状态的空对象，最终引发 `null` 值异常

这就是并发编程的极致复杂性。结合指令重排序，虽然出现这种情况的概率非常小，但是只要你的并发足够高，这基本上就可以看成是一个**必然事件**。所以，加 `volatile` 主要是为了避免读到空对象的情况。

------

## 五、延展讨论

**Q1：如果我们不采用引用替换的无锁方案，在这个场景下坚持使用锁机制，`ReentrantLock`、`ReentrantReadWriteLock` 或 `StampedLock` 该怎么选？有什么区别？**

**答：** 这三把锁都位于 `java.util.concurrent.locks` 包下，在这个一写多读、写耗时长的场景中表现各异：

1. **ReentrantLock（可重入互斥锁）**
   - **怎么用**：在 `get` 和 `load` 方法里都加上同一把互斥锁。
   - **优缺点**：绝对线程安全，实现简单。但它是**独占锁**。不仅读写互斥，致命的是**读读也互斥**。这会导致高并发下的缓存读取全部串行化，性能极差。在这个场景下极不推荐。
2. **ReentrantReadWriteLock（可重入读写锁）**
   - **怎么用**：在 `get` 中加读锁，`load` 中加写锁。
   - **优缺点**：实现了读读共存，读写互斥。在平时，大量线程可以同时读取，读性能很高。但它的致命缺陷在于**写锁阻塞读锁**：当定时任务执行 `load`（包含耗时的查库+装载数据，可能耗时几十毫秒）拿到写锁时，**所有的前端 `get` 读请求全部会被挂起阻塞**。这意味着系统每 5 分钟就会经历一次接口响应时间（RT）的严重毛刺。另外在高并发读的情况下，容易引发“写线程饥饿”。
   - **适用场景**：读多写少、写操作耗时极短（微秒级）的场景。
3. **StampedLock（邮戳锁 / 乐观读写锁）**
   - **怎么用**：JDK 8 引入的并发神器。在 `get` 时优先尝试乐观读（`tryOptimisticRead`，不加锁，拿到一个 stamp 戳），读完后校验戳是否失效（期间有没有发生写操作），如果失效再降级为悲观读锁。`load` 依然加排他写锁。
   - **优缺点**：**并发读取的性能天花板** 它的乐观读完全不阻塞写线程，有效解决了写饥饿。缺点是 API 极其复杂，而且**不可重入**，使用不当易引发死循环。但在我们的场景下，当 `load` 耗时较长导致乐观读验证失败并降级为悲观读锁时，依然会被挂起，还是无法彻底消除 RT 抖动。

**总结**：在当前这种 **定时全量刷新且包含耗时 IO** 的场景下，这三种锁都无法彻底避免前端请求的阻塞排队。因此，我们采用的 **“局部新空间装载 + volatile 引用瞬间替换”**（即 Copy-On-Write 思想）完美绕过了共享资源的竞争，是实现 0 阻塞、最高性能的最优解。

**Q2：HashMap 本身不是线程安全的，这里有没有必要改成 ConcurrentHashMap？**

**答：没必要。**

`ConcurrentHashMap` 主要解决的是并发写的安全问题。但在我们的场景中，真正的写操作（对 `newCache` 进行 `put`）永远只有定时任务这一个独立线程在局部变量上执行，压根不存在写并发的问题。而多线程读操作，读的是已经构建完毕的只读 Map，引用替换又是原子的。所以在这种“一写多读”且采用无锁替换的场景下，直接用普通的 `HashMap` 性能更好。

**Q3：这里的 CACHE 变量到底有没有必要用 static 修饰？**

**答：可以优化。**

由于这个 `ConfigCache` 类已经打上了 Spring 的 `@Component` 注解，由 Spring 容器管理的 Bean 默认就是单例的。既然类的实例在全局只有一份，那它的成员变量自然也只有一份，所以加不加 `static` 在功能上效果是一样的。去掉 `static` 会更符合面向对象的规范。

------

## 六、 最终优化后的代码

经过深度推演，我们既保证了高性能（无锁并发读），又保证了线程安全（防止指令重排序）的最终健壮代码如下：

```
@Component
public class ConfigCache {

    // 1. 去掉 static（依赖 Spring 单例）
    // 2. 加上 volatile 防止指令重排序引发的空对象泄露
    private volatile Map<String, String> cache = new HashMap<>();

    @Autowired
    private ConfigDAO configDAO;

    @PostConstruct
    public void init() {
        load();
    }

    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void reload() {
        load();
    }

    public String get(String key) {
        return cache.get(key); // 无锁高效读取
    }

    private void load() {
        List<ConfigDO> configs = configDAO.listAll();
        // 空间换时间：局部作用域构建新对象
        Map<String, String> newCache = new HashMap<>();
        for (ConfigDO config : configs) {
            newCache.put(config.getKey(), config.getValue());
        }
        // 原子操作，瞬间替换引用
        this.cache = newCache; 
    }
}
```

### 结语

AI 虽然能极快地帮我们搭起代码骨架，但它往往只关注功能实现，很难感知到复杂的业务上下文和 JVM 底层细节。一段看似极其简单的 CRUD 缓存代码，如果不深入推敲，就会踩中并发竞态条件和指令重排序的坑。

