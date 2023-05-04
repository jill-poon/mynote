# Microservices

[TOC]

## 演进中的架构

### 单体应用架构 (Monolithic)

* 项目**架构简单**，**开发、维护成本低**
* 缺点
  * 开发
    1. 大型项目**编译、运行、测试成本高**
    2. 项目**复杂度高**
    3. **技术栈**不易扩展
  * 部署
    1. 难以针对服务所需的**硬件性能**进行扩展
    2. 无法**故障隔离**，单个功能异常造成整个系统崩溃

### 垂直应用架构

* 实现了**流量分担**，解决**并发问题**
* 缺点
   1. 系统间相互调用复杂度提高 (配置 IP 端口)
   2. 系统间部分相同功能代码、数据冗余

### 分布式架构

* 公共功能为基础服务层
    1. **提高代码复用性**
    2. **降低系统间耦合度**
    3. **易于扩展**
* 缺点：
  * 随着服务数量增多，相互**调用关系复杂，难以维护**

### SOA 架构 (Service-Oriented Architecture)

* [3-SOA](3-SOA.md)
* 特点：
    1. **面向企业**
    2. 关注**冗余问题**
    3. 解决**信息孤岛**
* 缺点：
    1. 服务间有依赖关系，一旦某个环节出错会**引起服务雪崩**
    2. 服务**关系复杂，运维、测试、部署困难**

### 微服务架构 (Microservices)

* [微服务时代](https://icyfenix.cn/architecture/architect-history/microservices.html)
* 服务原子化拆分，**独立打包、部署、升级，利于扩展**
    1. **面向应用**
    2. 关注**服务粒度**
* 缺点
  * **技术**成本高
    * 容错
    * 分布式事务
  * 服务**治理**成本高
* 与 SOA 区别
    1. SOA 是一种设计方法，**关注服务重用性**，解决**企业内部**的**信息孤岛**问题
    2. 微服务**关注解耦**、使用**更轻量级通信协议**、更多**关注 DevOps 持续交互**
    3. SOA 是微服务的**超集**

### 后微服务时代 （Cloud Native）

* [后微服务时代](https://icyfenix.cn/architecture/architect-history/post-microservices.html)

### 无服务时代 (Serverless)

* [无服务时代](https://icyfenix.cn/architecture/architect-history/serverless.html)

## 通信协议

### RPC

* 强调像调用本地方法一样调用远程服务
* 强调网络通信
* 强调序列化/反序列化

### HTTP

* 超文本协议
* OSI七层模型的应用层协议
* 数据传输协议

## 微服务架构目的

* 高可用
* 高并发
* 高性能
* 高可扩展

## [必要条件](10-必要条件.md)

### 1 DevOps Experience

[DevOps](5-DevOps.md)

### 2 弹性扩容 (Auto Scaling & Self Healing)

### 3 弹性和容错 (Resilence & Fault Tolerance)

### 4 链路追踪 (Distributed Tracing)

* Skywalking
  * 监控粒度细
* CAT
* zipkin
* pingpoint

### 5 系统监控 (Centralized Metrics)

* ELK (完善)
* [Prometheus](metrics/Prometheus/Prometheus.md) + grafana
* Zabbix
* Falcon
* [Prometheus vs ELK](metrics/Prometheus/Prometheus-vs-ELK.md)

### 6 Centralized Logging

### 7 API 网关 (API Gateway)

* 网关
  * 流量网关
  * 业务网关
* 统一服务入口
* 流量治理

### 8 任务管理 (Job Management)

### 9 Singleton Application

### 10 负载均衡 (Load Balancing)

### 11 服务发现 (Service Discovery)

1. 服务注册
2. 服务续约
3. 服务获取
4. 服务调用
5. 服务同步
6. 服务剔除
7. 自我保护
8. 服务下线
9. 动态感知

### 12 配置中心 (Configuration Management)

* zk
* etcd
* nacos
* consul

### 13 Application Packaging

### 14 Deployment & Scheduling

### 15 进程隔离 (Process Isolation)

### 16 环境管理 (Environment Management)

### 17 Resource Management

### 18 Operating System

### 19 Virtualization

### 20 Hardware, Storage, Networking
