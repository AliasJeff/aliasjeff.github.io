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

* 熟练掌握 Java 语言，如集合类、注解、异常、动态代理、AOP、SPI机制；
* 熟悉并发编程，掌握Java线程池实现原理、synchronized、volatile等；
* 熟悉 Mysql，熟悉索引、事务、锁机制，参与过慢 sql 治理；
* 熟练掌握 Redis ，实践过分布式缓存、分布式锁，熟悉持久化原理，了解主从集群、哨兵等原理；
* 熟悉 SpringBoot 核心知识，熟悉 Java 常用类库，了解 IoC、AoP 基本思想；
* 了解 Spring AI 框架，了解 RAG、Agent、MCP、Skill 等技术，能够运用 AI 工具为业务提效赋能；
* 熟悉常用 Linux 命令，熟练使用 Git、Navicat、Maven、Docker 等开发工具、项目管理工具。

---

## 💼 工作经历

**Taelor Inc.** (AI 驱动的男装租赁订阅平台) | **后台开发实习生** | **2023.06 - 2025.06**

**工作内容**：负责营销抽奖模块的架构设计与核心功能开发，支持多种抽奖策略、负责规则配置、高并发库存安全控制；设计并实现顾客评价系统，支持多级评论与高并发点赞场景。

1. 负责营销服务的抽奖模块开发，支持各类差异化抽奖流程，运用多种设计模式，通过责任链 + 决策树的规则引擎统一编排整个抽奖策略流程。对抽奖的计算和秒杀做了优化，经测试可以支撑 1200 ~ 1500 TPS 的吞吐量，抽奖接口响应时长 45 毫秒左右。
2. 在奖品库存扣减上，采用了 Redis decr 原子扣减和滑块锁兜底的设计，防止超卖；使用延迟队列 + 定时任务更新库存，削峰填谷，减轻数据库的压力。
3. 针对商品评价、多级评论与点赞等高频交互场景，采用模板方法与策略模式抽象评价处理流程，并结合 Redis 实现点赞去重与计数缓存，支持高并发访问与平滑扩展。
4. 针对长事务、锁等待等问题，基于 AOP 与 TransactionSynchronizationManager 在事务生命周期层实现无侵入监控，精确采集最外层事务耗时与异常上下文，并异步审计落库，实现事务级可观测与问题可追溯性。

---

## 💻 项目经验

**仿PDD的高并发拼单交易系统** | <u>[Github 链接](https://github.com/AliasJeff/pin-mate-buy)</u>

**核心技术**：SpringBoot、MyBatis、MySQL、Redis、RabbitMQ、Guava、XXL-Job、Docker

**项目描述**：本项目参考拼多多、拼好饭等电商拼单交易系统，实现了 优惠试算 → 锁单 → 支付结算 → 退单 的全链路业务流程。模拟大促活动期间流量大、规则复杂的情况，通过对拆分业务模块、引入缓存机制以及异步处理方案，解决了高并发下的抢单卡顿、库存不足以及系统响应慢等问题。

1. 高并发库存安全 (防超卖)：锁单环节采用 Redis 分段锁，拆分库存粒度，降低数据库行锁竞争，提高系统并发量，在保证高性能的同时防止超卖。
2. 一致性保障：采用 本地消息表 + RabbitMQ 的组合方案。在业务事务提交的同时，将待发送的消息记录落库，由定时任务和异步任务扫描本地消息表并发送至 MQ，确保消息必达和拼单状态最终一致性。
3. 动态配置管理：设计基于 Redis Pub/Sub 模型的动态配置中心，结合 Spring AOP 切面和代理，以自定义注解的方式控制属性信息动态配置，实现活动开关、降级策略的秒级热更新（无需重启）。

**RAG 驱动的智能代码审查助手（AI Code Review Agent）** | <u>[Github 链接](https://github.com/AliasJeff/alias-rag-review)</u>

**核心技术**：Spring AI、RAG、Agent、PgVector、Tool Calling、SSE、React

**项目描述**：本项目基于 RAG 构建代码知识库，结合 ReAct Agent 工作流实现代码上下文检索、LLM 推理与自动决策，并通过 GitHub API 自动发布 Code Review 评论。提供对话式前端与 SSE 流式响应，支持多轮交互式代码审查。

1. 实现 RAG 知识库构建能力，支持文件上传与 Github 仓库导入，完成文档切分、Embedding 向量化与 PgVector 索引(HNSW) 构建，基于标签让 Advisor 精准访问知识库，提升任务执行的准确率。
2. 构建多步骤 Agent 工作流（ReAct + Tool Calling），结合代码上下文检索 → LLM 推理 → 自动决策 → Review 评论生成，实现智能决策与执行闭环。
3. 开发对话式交互界面与 SSE 流式响应，允许开发者与 AI Reviewer 多轮沟通，深入理解审查建议与优化策略。

---

## 🏆 荣誉奖项

* **国家级**：第七届“互联网+”大学生创新创业大赛 **全国金奖** (2021)
* **国家级**：第十三届“挑战杯”大学生创业计划竞赛 **全国银奖** (2023)
* **国家级**：第十五届服务外包大赛 **全国铜奖** (2024)
* **省级**：连续两年获得 **浙江省政府奖学金** (2021-2023)
* **校级**：连续两年获得 **校一等奖学金** (2021-2023)

---

# Zhexun (Jeffery) Chen

📞 (+86) 195-5786-4422 | 📧 zhexunchen@gmail.com
💻 [Github: AliasJeff](https://github.com/AliasJeff/) | 📝 [Blog: alias-studio.github.io](https://alias-studio.github.io/)

---

## 🎓 Education

**Xidian University** | *M.S. in Software Engineering* | **Aug 2025 - Present**

**Zhejiang Normal University** | *B.S. in Software Engineering* | **Oct 2020 - Jun 2025**

* **Academics:** Top 5% ranking; Received **2 National Awards** (including "Internet+" Gold Award); Published 2 Software Copyrights.
* **Awards:** Provincial Government Scholarship (2021-2023); University First-Class Scholarship (2021-2023).
* **Certifications:** CET-6 (Score: 579); Intermediate Software Designer; National Computer Rank Examination Grade 2.

---

## 🛠 Professional Skills

* **Java Core:** Proficient in Java, including Collections, Annotations, Exceptions, Dynamic Proxies, AOP, and SPI mechanisms.
* **Concurrency:** Familiar with Java concurrency, underlying implementations of Thread Pools, `synchronized`, `volatile`, etc.
* **Database:** Familiar with **MySQL** indexing, transactions, and locking mechanisms; experienced in Slow SQL tuning and governance.
* **Middleware:** Proficient in **Redis** (Distributed Cache/Locks, Persistence, Master-Slave/Cluster/Sentinel architectures).
* **Frameworks:** Solid understanding of **Spring Boot** core concepts, Java standard libraries, and IoC/AOP principles.
* **AI Technologies:** Familiar with the **Spring AI** framework, **RAG, Agent, MCP, and Skill** technologies; capable of utilizing AI tools to empower and optimize business workflows.
* **Tools & DevOps:** Proficient in common Linux commands, Git, Navicat, Maven, and Docker.

---

## 💼 Work Experience

**Taelor Inc.** (AI-Powered Menswear Rental Subscription) | **Backend Development Intern** | *Jun 2023 - Jun 2025*

**Overview:** Responsible for the architecture design and core development of the marketing lottery module, supporting diverse lottery strategies, rule configurations, and high-concurrency inventory safety control. Designed and implemented a customer evaluation system supporting multi-level comments and high-concurrency "like" scenarios.

* **Marketing & Lottery Module:** Developed the marketing lottery service supporting differentiated workflows. Orchestrated the entire strategy using the **Chain of Responsibility + Decision Tree** rule engine. Optimized calculation and flash-sale logic, achieving **1,200 ~ 1,500 TPS** throughput with an average response time of ~**45ms**.
* **Inventory Control:** Implemented **Redis `decr` + Slider Lock** as a fallback mechanism to prevent overselling. Utilized a **Delay Queue + Scheduled Tasks** for asynchronous database inventory updates, effectively smoothing traffic peaks and reducing DB pressure.
* **Evaluation System:** Abstracted the evaluation process for high-frequency interactions (multi-level comments, likes) using the **Template Method & Strategy patterns**. Integrated Redis for "like" deduplication and aggregated counting, ensuring high concurrency support and smooth scalability.
* **Transaction Observability:** Addressed long-transaction and lock-waiting issues by building a non-intrusive monitoring layer based on **AOP & `TransactionSynchronizationManager`**. Accurately captured outer-transaction execution times and exception contexts, storing them asynchronously for transaction-level observability and traceability.

---

## 💻 Project Experience

**High-Concurrency Group-Buying Trading System (PDD-like)** | <u>[Github Link](https://github.com/AliasJeff/pin-mate-buy)</u>

**Tech Stack:** Spring Boot, MyBatis, MySQL, Redis, RabbitMQ, Guava, XXL-Job, Docker.

**Description:** Modeled after e-commerce platforms like Pinduoduo, realizing the full business lifecycle: Discount Calculation → Order Locking → Payment → Refund. Simulated high-traffic promotional events, solving issues like order-grabbing lag, inventory shortages, and slow responses via module splitting, caching, and async processing.

* **Inventory Safety (Anti-overselling):** Implemented **Redis Segmented Locks** during the order-locking phase to reduce database row lock contention, significantly improving system concurrency while guaranteeing zero overselling.
* **Consistency Guarantee:** Adopted the **"Local Message Table + RabbitMQ"** pattern. Synchronously persisted pending messages during business transaction commits, and utilized scheduled/async tasks to push messages to MQ, ensuring reliable delivery and eventual consistency of group-buying statuses.
* **Dynamic Configuration:** Built a dynamic configuration center using the **Redis Pub/Sub** model. Combined with Spring AOP and custom annotations, enabled **second-level hot updates** (without restarts) for promotional toggles and downgrade strategies.


**RAG-Driven AI Code Review Agent** | <u>[Github Link](https://github.com/AliasJeff/alias-rag-review)</u>

**Tech Stack:** Spring AI, RAG, Agent, PgVector, Tool Calling, SSE, React.

**Description:** Built a code knowledge base using RAG, integrated with a ReAct Agent workflow for context retrieval, LLM reasoning, and automated decision-making. Automatically posts Code Review comments via the GitHub API. Features a conversational UI with SSE stream responses for multi-turn interactive reviews.

* **RAG Knowledge Base:** Supported file uploads and GitHub repository imports. Implemented document chunking, Embedding vectorization, and **PgVector HNSW index** construction. Enabled precise knowledge base access via tags to improve the Agent's execution accuracy.
* **Agent Workflow:** Developed a multi-step Agent workflow (**ReAct + Tool Calling**) bridging code context retrieval → LLM reasoning → automated decision → Review comment generation, creating a closed-loop intelligent review process.
* **Conversational UI:** Built an interactive frontend with **SSE streaming**, allowing developers to engage in multi-turn dialogues with the AI Reviewer to deeply understand review suggestions and optimization strategies.

---

## 🏆 Honors & Awards

* **National Gold Award**, 7th China International College Students' "Internet+" Innovation Competition (2021)
* **National Silver Award**, 13th "Challenge Cup" National College Students' Entrepreneurship Competition (2023)
* **National Bronze Award**, 15th National College Student Service Outsourcing Innovation and Entrepreneurship Competition (2024)
* **Provincial Government Scholarship**, Zhejiang Province (2021-2023)
* **University First-Class Scholarship** (2021-2023)