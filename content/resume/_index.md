+++
title = "Résumé"

[extra]
comment = true
+++

# 陈喆勋

📞 19557864422 | 📧 zhexunchen@gmail.com
💻 [Github: AliasJeff](https://github.com/AliasJeff/) | 📝 [Blog: alias-studio.github.io](https://alias-studio.github.io/)

---

## 🎓 教育经历

**西安电子科技大学** | 软件工程 | 硕士 | *2025.08 - 至今*
**浙江师范大学** | 软件工程 | 本科 | *2020.10 - 2025.06*

* **荣誉奖项**：综合成绩专业 **前5%**；获“互联网+”等 **国家级奖项 2 项**；发表软著 2 篇；多次获省政府/校级一等奖学金。
* **证书**：英语六级 (579)；中级软件设计师；计算机二级。

---

## 🛠 专业技能

* **Java 核心**：精通 Java 基础及集合框架；熟练掌握注解、反射、动态代理、SPI 机制；理解 AOP/IoC 思想。
* **并发编程**：深入理解 JUC，掌握线程池原理、synchronized/volatile 关键字底层机制、锁升级过程及 AQS 原理。
* **数据库**：熟练掌握 MySQL 索引优化、事务隔离级别及锁机制；具备慢 SQL 治理与生产环境调优经验。
* **分布式/中间件**：
  * **Redis**：熟练运用分布式锁、缓存一致性策略；理解 RDB/AOF 持久化、哨兵及集群模式原理。
  * **RabbitMQ**：掌握消息持久化、TTL、死信队列、延迟队列及消息可靠性投递方案。
  * **RPC/微服务**：熟悉 Dubbo 框架及负载均衡、服务容错（重试/降级）机制。
* **工具与生态**：熟练使用 Spring Boot, MyBatis, Maven, Git, Docker, Linux 常用命令。

---

## 💼 工作经历

**Taelor Inc.** (AI 驱动的男装租赁订阅平台) | **后台开发实习生** | **2023.06 - 2025.06**

**工作内容**：负责营销服务架构设计，涵盖抽奖、积分、优惠券体系；设计高并发评论评价系统；主导线上故障排查与性能治理。

* **营销抽奖系统重构**：针对差异化抽奖流程运用策略模式与模板方法重构代码；优化秒杀算法，经压测系统吞吐量提升至 **1200~1500 TPS**，接口响应稳定在 **45ms**。
* **高并发库存防超卖**：设计**“Redis Decr 分段扣减 + 异步队列 + 定时任务同步”**的多级库存扣减方案，有效解决数据库行锁瓶颈，确保高并发下的**库存一致性**。
* **评价系统优化**：针对多级评论与高频点赞场景，结合 Redis 实现点赞去重与计数**聚合缓冲**，支撑高并发访问；通过**策略模式**解耦复杂评价规则，提升代码可维护性。
* **事务级可观测性建设**：针对长事务与锁等待痛点，基于 AOP 与 `TransactionSynchronizationManager` 实现**无侵入式**事务监控，精确采集最外层事务耗时与异常上下文并异步落库，提升问题排查效率。
* **线上稳定性保障**：参与解决 **10+** 起线上故障（慢 SQL、长事务等），并输出完整的技术复盘文档与开发规范。

---

## 💻 项目经验

**高并发电商交易系统 (拼单/秒杀)** | [Github 链接](https://github.com/AliasJeff/alias-rpc)

**核心技术**：SpringBoot, MySQL, Redis, RabbitMQ, Guava, XXL-Job, Docker, DDD

**项目描述**：集多人拼团、限时秒杀、优惠营销于一体的电商交易平台，实现了从优惠试算、锁单、支付到退单的全链路闭环。

* **DDD 与规则引擎设计**：基于 DDD 四色建模构建业务，配合**规则树** + **责任链**模式，将人群筛选、限购校验、优惠试算等复杂逻辑**模块化**，实现了业务规则的**动态编排**与**热插拔**。
* **高并发库存安全 (防超卖)**：锁单环节采用 Redis **分段锁**，拆分库存粒度，降低数据库行锁竞争，提高系统**并发量**，在保证高性能的同时防止超卖。
* **一致性保障**：采用 **本地消息表 + RabbitMQ** 的组合方案。在业务事务提交的同时，将待发送的消息记录**落库**，由**定时任务**和**异步任务**扫描本地消息表并发送至 MQ，确保**消息必达**和拼单状态**最终一致性**。
* **动态配置管理**：设计基于 Redis Pub/Sub 模型的**动态配置中心**，结合 **Spring AOP 切面和代理**，以**自定义注解**的方式控制属性信息**动态配置**，实现活动开关、降级策略的**秒级**热更新（无需重启）。



**Alias-RPC (可扩展的高性能 RPC 框架)** | [Github 链接](https://github.com/AliasJeff/alias-rpc)

**核心技术**：Netty, Vert.x, Etcd, SPI, Spring Boot Starter, 自定义协议

**项目描述**：基于 Etcd 与 Vert.x 构建的轻量级 RPC 框架，支持注解式调用，具备高度可扩展性。

* 实现了**高可用**的**分布式注册中心**，利用其层级结构和 Jetcd 的 KvClient 存储服务和节点信息，并支持 SPI 机制扩展。
* 利用定时任务和 Etcd Key 的 TTL 实现服务提供者的**心跳检测**和**续期机制**，节点下线一定时间后**自动移除**注册信息。
* 基于 Vert.x 的 RecordParse 完美解决**半包粘包**问题，使用**装饰者模式**封装了 TcpBufferHandlerWrapper 类，一行代码即可对原有的请求处理器进行增强，提高代码可维护性。
* 使用 Jmeter 进行压力测试，使用 5000 个线程并发调用，TPS 达到 20000 左右。

---

## 🏆 荣誉奖项

* **国家级**：第七届“互联网+”大学生创新创业大赛 **全国金奖** (2021)
* **国家级**：第十三届“挑战杯”大学生创业计划竞赛 **全国银奖** (2023)
* **国家级**：第十五届服务外包大赛 **全国铜奖** (2024)
* **省级**：连续两年获得 **浙江省政府奖学金** (2021-2023)
* **校级**：连续两年获得 **校一等奖学金** (2021-2023)

---



# Zhexun(Jeffery) Chen

📞 (+86) 195-5786-4422 | 📧 zhexunchen@gmail.com
💻 [Github: AliasJeff](https://github.com/AliasJeff/) | 📝 [Blog: alias-studio.github.io](https://alias-studio.github.io/)

---

## 🎓 Education

**Xidian University** | *M.S. in Software Engineering* | **Aug 2025 - Present**
**Zhejiang Normal University** | *B.S. in Software Engineering* | **Oct 2020 - Jun 2025**

* **Academics:** Top 5% ranking; Received **National Gold Award** in "Internet+" Competition; Published 2 Software Copyrights.
* **Awards:** Provincial Government Scholarship (Twice); University First-Class Scholarship (Twice).
* **Certifications:** CET-6 (Score: 579); Intermediate Software Designer.

---

## 🛠 Professional Skills

* **Java Core:** Proficient in Java SE, Collections, Reflection, and SPI; Deep understanding of **AOP/IoC** principles and Dynamic Proxies.
* **Concurrency:** Mastered **JUC** (java.util.concurrent), Thread Pools, `synchronized`/`volatile` internals, Lock Escalation, and AQS mechanisms.
* **Database:** Expert in **MySQL** indexing, transaction isolation levels, and locking mechanisms; Experienced in Slow SQL tuning and production optimization.
* **Distributed Systems:**
  * **Redis:** Proficient in Distributed Locks, Cache Consistency, Persistence (RDB/AOF), and Cluster/Sentinel architectures.
  * **RabbitMQ:** Skilled in message persistence, TTL, Dead Letter Queues, and ensuring reliable message delivery.
  * **RPC/Microservices:** Familiar with Dubbo, Load Balancing algorithms, and Fault Tolerance strategies (Retry/Fallback).
* **Tools & DevOps:** Spring Boot, MyBatis, Maven, Git, Docker, Linux, Navicat.

---

## 💼 Work Experience

**Taelor Inc.** (AI-Powered Menswear Rental Subscription) | **Backend Development Intern** | *Jun 2023 - Jun 2025*

**Overview:** Responsible for the architecture of the Marketing Service (Lottery/Points/Coupons) and the Evaluation System. Led stability governance initiatives including Slow SQL tuning and Long Transaction optimization.

* **Marketing System Refactoring:** Redesigned the lottery module using **Strategy & Template Method patterns** to handle differentiated workflows. Optimized the flash-sale algorithm, achieving **1,200–1,500 TPS** with stable response times around **45ms**.
* **Inventory Concurrency Control:** Engineered a "Redis Decr + Async Queue + Scheduler" mechanism to prevent **overselling**, significantly reducing database row lock contention under high concurrency.
* **Evaluation System Optimization:** Developed a high-concurrency comment/like system. Implemented Redis for deduplication and aggregated counting; decoupled complex business rules using Strategy patterns to ensure scalability.
* **Transaction Observability:** Addressed long-transaction issues by building a **non-intrusive monitoring system** based on AOP and `TransactionSynchronizationManager`. Achieved precise logging of transaction execution time and exception contexts.
* **Stability Governance:** Resolved **10+** critical production incidents (e.g., Slow SQLs, Deadlocks) and authored technical documentation and development standards.

---

## 💻 Project Experience

**High-Concurrency E-Commerce System** | [Github Link](https://github.com/AliasJeff/alias-rpc)

* **Tech Stack:** Spring Boot, MySQL, Redis, RabbitMQ, Guava, XXL-Job, Docker, DDD.
* **Description:** A comprehensive trading platform integrating Group Buying and Flash Sales. Implemented the full lifecycle from discount calculation, inventory locking, and payment, to refunds.
* **DDD & Rule Engine:** Modeled business boundaries using **DDD**; Implemented a "Rule Tree + Chain of Responsibility" engine to dynamically orchestrate user filtering and complex discount rules.
* **Inventory Safety:** Implemented **Redis Segmented Locks** to split inventory granularity, maximizing concurrency performance while guaranteeing zero overselling.
* **Distributed Consistency:** Adopted the **"Local Message Table + RabbitMQ"** pattern. utilized scheduled tasks and async compensation to ensure Eventual Consistency between trade data and message delivery.
* **Dynamic Configuration:** Built a configuration center based on **Redis Pub/Sub** and Spring AOP, enabling **second-level hot updates** for feature toggles and downgrade strategies without service restarts.

**Alias-RPC Framework** | [Github Link](https://github.com/AliasJeff/alias-rpc)

* **Tech Stack:** Netty, Vert.x, Etcd, SPI, Spring Boot Starter, Custom Protocol.
* **Description:** A high-performance, extensible RPC framework built on Etcd and Vert.x, supporting annotation-based remote calls.
* **High Performance:** Designed a custom communication protocol over Netty/TCP. Solved TCP Packet Sticking/Splitting issues using `RecordParser`. Achieved **20,000+ QPS** under 5,000 concurrent threads.
* **SPI Extensibility:** Implemented **SPI (Service Provider Interface)** to allow dynamic plugging of Serializers (JDK/JSON/Kryo), Load Balancers, and Fault Tolerance strategies.
* **Service Governance:** Integrated **Etcd** for service registry/discovery. Implemented Keep-Alive mechanisms using Key TTL. Enhanced request handling logic via the Decorator pattern (`TcpBufferHandlerWrapper`).

---

## 🏆 Honors & Awards

* **National Gold Award**, 7th China International College Students' "Internet+" Innovation Competition (2021)
* **National Silver Award**, 13th "Challenge Cup" National College Students' Entrepreneurship Competition (2023)
* **National Bronze Award**, 15th Service Outsourcing Innovation & Entrepreneurship Competition (2024)
* **Provincial Government Scholarship** (Awarded in 2021-2022 & 2022-2023)

