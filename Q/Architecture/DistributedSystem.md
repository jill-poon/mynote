# 分布式系统

[分布式系统](../../Architecture/20-DistributedSystem/0-分布式系统.md)

## 分布式系统特点

> 核心概念：\_\_\_\_\_\_

### 1 \_\_

- 在用户面前的表现就想一个\_\_，用户不必了解其内部就够就能使用

### 2 \_\_

- 能\_\_，可通过\_\_集群使集群的\_\_

### 3 \_\_

- \_\_，具有持续服务的特性

### 4 \_\_

- \_\_
- 缺点
    1. 在节点\_\_， 并发安全问题也变得复杂， 需要在保证数据完整性的同时兼顾性能
    2. \_\_， 网络信息的丢失和饱和将会抵消分布式系统的大部分优势
    3. 有潜在的\_\_和\_\_等安全性问题

## 什么是 CAP 原则

- \_\_。 分布式系统中\_\_在\_\_是否\_\_。
- \_\_。 在集群中\_\_后，\_\_是否还能\_\_客户端读写\_\_。
- \_\_。 对通信的**时限要求**。分布式系统在遇到\_\_的时候，仍然能够\_\_。

## 什么是BASE原则

- \_\_。分布式系统出现\_\_，\_\_。
  - 保证核心服务可用：降低响应时间，甚至服务降级。
- \_\_。允许系统中数据存在\_\_，并认为该中间状态\_\_。*允许系统不同节点的数据副本之间\_\_*
- \_\_。强调\_\_，经过一段时间同步后，\_\_的状态。
