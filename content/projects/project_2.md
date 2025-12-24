+++
title = "可扩展的高性能 RPC 框架"
description = "自研基于 Vert.x 的高性能 RPC 通信框架，支持动态代理、服务注册发现、负载均衡与容错扩展。"
weight = 3

[extra]
local_image = ""
github = "https://github.com/AliasJeff/alias-rpc"
tags = ["RPC", "SPI", "注册中心", "负载均衡", "设计模式", "分布式"]
+++

**核心技术**：Spring Boot、Vert.x、Etcd、SPI、自定义协议、Netty/TCP、Docker

**项目描述**：基于 Etcd 与 Vert.x 构建的高性能、可扩展 RPC 框架，通过自定义通信协议与 Spring Boot Starter 方式接入，支持以**注解**和**配置**的形式像调用本地方法一样调用远程服务，并支持通过**SPI**机制**动态扩展**序列化器、负载均衡、重试和容错策略等。

1. 实现了**高可用**的**分布式注册中心**，利用其层级结构和 Jetcd 的 KvClient 存储服务和节点信息，并支持 SPI 机制扩展。
2. 利用定时任务和 Etcd Key 的 TTL 实现服务提供者的**心跳检测**和**续期机制**，节点下线一定时间后**自动移除**注册信息。
3. 基于 Vert.x 的 RecordParse 完美解决**半包粘包**问题，使用**装饰者模式**封装了 TcpBufferHandlerWrapper 类，一行代码即可对原有的请求处理器进行增强，提高代码可维护性。
4. 使用 Jmeter 进行压力测试，使用 5000 个线程并发调用，TPS 达到 20000 左右。

