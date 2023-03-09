# Kubernetes

K8s 是一个容器编排系统，其关注的是容器应用在整个集群的管理和部署

## 架构

![kubernetes-architecture](images/kubernetes-architecture.drawio.svg)

### Master Node

#### 1 API Server

- `kube-apiserver`
- 用户唯一可以进行交互的组件
- 控制程序的前端
- 内部系统组件以及外部用户组件均通过相同 API 进行通讯

#### 2 etcd

- `etcd`
- 备份所有集群数据的数据库
- 存储集群整个配置和状态
- 主节点查询 etcd，以检索节点，容器和容器的状态参数

#### 3 Controller Manager

- `kube-controller-manager`
- 作用是从 `API Server` 获得所需状态
- 检查要控制的节点的当前状态，确定是否与所需状态存在任何差异，并解决他们

#### 4 Scheduler

- `kube-scheduler`
- 调度程序会监视来自 `API Server` 的新请求，并将其分配给运行状况良好的节点
- 对节点的质量排名，将 `Pod` 部署到最合适的节点
- 没有合适的节点，将 `Pod` 置于挂起状态，直到出现合适的节点

### Workder Node

#### 1 Kubelet

- 在集群上**每个节点运行**
- **主要代理**
- 节点的 CPU、RAM 和存储成为所处集群的一部分
- 监视从 `API Server` 发送来的任务，执行任务，并报告给主节点
- 监视 `Pod`，如果 `Pod` 不能完全正常运行，会向控制程序报告
- 基于该信息，主服务器可以决定如何分配任务和资源以达到所需状态

#### 2 Container Runtime

- CRI (Container Runtime Interface)
- 拉取镜像
- 启动和停止容器

#### 3 Network Proxy

- kube-proxy
- 维护节点上的网络规则，实现了 Kubernetes Service 概念的一部分
- 它的作用是使发往 Service 的流量（通过 ClusterIP 和端口）负载均衡到正确的后端 `Pod`
- 监听 API server 中资源对象的变化情况，service、endpoint/endpointslices、node，然后**根据监听资源变化**操作代理后端来**为服务配置负载均衡**
    1. userspace （userspace 是早期版本的实现）
    2. iptables
    3. ipvs （Linux内核功能）
    4. kernelspace（专用于windows）

#### 4 Pod

- pause (Infra container)
  - `Pod` 内所有业务容器的根容器
  - 容器之间原本是被 Linux Namespace 和 cgroups 隔开的，根容器实现 Pod 内所有容器间的网络和存储共享
  - 整个 Pod 的生命周期是等同于 Infra container 的生命周期

#### 5 CoreDNS

- 解析域名为 `${service_name}.${namespace}.svc.cluster.local`
- 其中 cluster.local代表集群的后缀
- [CoreDNS调优](https://github.com/coredns/deployment/blob/master/kubernetes/Scaling_CoreDNS.md)

#### 6 Optional Add-Ons

- DNS
- UI

## 核心对象

### Namespace

- 将对象逻辑分配到不同命名空间
- 可以是不同项目、用户等区分管理
- 可根据不同命名空间**设定控制策略**
- 虚拟集群

### Pod

- **最小的部署单元**
- 一个 `Pod` 由一个或多个容器组成
- `Pod` 中容器共享存储和网络，在同一个工作节点上运行

### Service

- 一个应用服务抽象，定义了 `Pod` 逻辑集合和访问这个 `Pod` 集合的策略
- `Pod` 集合**对外访问入口**，分配一个集群 IP 地址，来自这个 IP 的请求将负载均衡转发后端`Pod` 中容器
- 通过 Lable Selector 选择一组 `Pod` 提供服务
- 当 Pod 宕机后重新生成时，其 IP 等状态信息可能会变动，Service 会根据 Pod 的 Label 对这些状态信息进行监控和变更，保证上游服务不受 Pod 的变动而影响

#### 1 ClusterIP

- 默认类型
- 自动分配一个仅 Cluster 内部可以访问的虚拟 IP

#### 2 NodePort

- 在 ClusterIP 基础上为 Service 在每台机器上绑定一个端口
- 这样就可以通过 NodePort 来访问该服务

#### 3 LoadBalancer

- 在 NodePort 的基础上，借助 Cloud Provider 创建一个外部负载均衡器，并将请求转发到 NodePort

#### 4 ExternalName

- 把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建
- 只有 Kubernetes 1.7或更高版本的 kube-dns才支持。

### ReplicaSet

- 下一代 Replication Controller
- 确保任何给定时间指定的 `Pod` 副本数量，并并提供声明式热更新等功能
- RC 与 RS 唯一区别是 lable selector 支持不同，RS 支持新的基于集合的标签，RC仅支持基于等式的标签

### Deployment

- 管理 ReplicaSet 和 `Pod` ，提供声明式更新等功能
- 官方建议 Deployment 管理 ReplicaSet，而不是直接使用 ReplicaSet

### StatefuleSet

- 适合持久性的应用程序
- 有唯一网络标识符 (IP)
- 持久储存
- 有序的部署
- 扩展
- 删除
- 滚动更新

### DaemonSet

- 确保所有节点运行同一个 Pod

### Job

- 一次性任务，完成后 `Pod` 销毁
  
### Volume

数据卷

#### emptyDir

- **生命周期同 `Pod`**
- 应用场景
  - **临时空间**，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
  - 长时间任务的中间过程 CheckPoint 的临时保存目录
  - 一个容器需要从另一个容器中获取数据的目录(多容器共享目录)

#### hostPath

- 在 `Pod` 上挂载宿主机上的文件或目录
- 应用场景
  - 容器应用程序生成的日志文件需要永久保存时，可以使用宿主机的高速文件系统进行存储
  - 需要访问宿主机上 Docker 引擎内部数据结构的容器应用时，可以通过定义 hostPath 为宿主机 /var/lib/docker 目录，使容器内部应用可以直接访问 Docker 的文件系统

#### nfs

### PersistentVolume

是集群内，由管理员提供的网络存储的一部分

### StorageClass

- 作为对存储资源的抽象定义
- 对用户设置的 PVC 申请屏蔽后端存储的细节
  - 一方面减少用户对存储资源细节的关注
  - 另一方面减少管理员手工管理 PV 的工作
- 由系统自动完成 PV 的创建和绑定，实现**动态资源**供应

### PersistentVolumeClaim

- 持久卷声明
- 它和 `Pod` 类似，`Pod` 消耗 Node 资源，而 PVC 消耗 PV 资源
- PVC 与 PV 的绑定是一对一的映射。没找到匹配的 PV，那么 PVC 会无限期得处于 unbound 未绑定状态
- volumeClaimTemplates
  - PVC 的模板，指定 PVC 名称大小，自动创建 PVC，且 PVC 必须由存储类供应

### ConfigMap

- ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中

- `Pod` 可以将其用作环境变量、命令行参数或者存储卷中的配置文件

## 概念

### Qos

- QoS(Quality of Service) 即服务质量
- QoS 是**一种控制机制**，它提供了针对不同用户或者不同数据流采用相应不同的优先级
- 或者是根据应用程序的要求，保证数据流的性能达到一定的水准
- kubernetes 中有三种 Qos，分别为：
  - Guaranteed：pod 的 requests 与 limits 设定的值相等
  - Burstable：pod requests 小于 limits 的值且不为 0
  - BestEffort：pod 的 requests 与 limits 均为 0
- 三者的优先级如下所示，依次递增
  - `BestEffort -> Burstable -> Guaranteed

#### 不同 Qos 的本质区别

- 在调度时调度器只会根据 request 值进行调度；
- 当系统 OOM 上时对于处理不同 OOMScore 的进程表现不同，也就是说当系统 OOM 时，首先会 kill 掉 BestEffort pod 的进程，若系统依然处于 OOM 状态，然后才会 kill 掉 Burstable pod，最后是 Guaranteed pod

下面的表格中反应了 `kube-state-metrics` 提供的4个相关指标。

| 指标名称                                          | 含义                     | 单位说明                       |
| ------------------------------------------------- | ------------------------ | ------------------------------ |
| kube_pod_container_resource_requests_cpu_cores    | 容器设置的cpu requests值 | request=100m 代表使用0.1个核心 |
| kube_pod_container_resource_requests_memory_bytes | 容器设置的mem requests值 | 单位：字节                     |
| kube_pod_container_resource_limits_cpu_cores      | 容器设置的cpu limits值   | request=100m 代表使用0.1个核心 |
| kube_pod_container_resource_limits_memory_bytes   | 容器设置的 mem limits 值 | 单位：字节                     |

#### CPU 属于可压缩资源

- CPU 属于可压缩资源
- `Pod` 中服务使用 CPU 超过设置的 limits 不会被 kill 掉但会被限制
- **应该通过观察容器 CPU 被限制的情况来考虑是否将 CPU 的 limit 调大**

#### CPU 限制率和利用率

##### 限制率

- 有这样的两个 CPU 指标，`container_cpu_cfs_periods_total` 代表 container 生命周期中度过的 CPU 周期总数
- `container_cpu_cfs_throttled_periods_total` 代表 container 生命周期中度过的受限的 CPU 周期总数。
- 所以我们可以使用下面的表达式来查出最近 5 分钟，超过 25% 的 CPU 执行周期受到限制的 container 有哪些。

```shell
 100 *  sum by(container_name, pod_name, namespace) (increase(container_cpu_cfs_throttled_periods_total{container_name!=""}[5m]))
  / sum by(container_name, pod_name, namespace) (increase(container_cpu_cfs_periods_total[5m]))  > 25
```

- 如果上述 promql 有查询结果，我们可以考虑将 CPU 的 limit 调大

##### 利用率

- 可以用下面的计算方式表示容器 CPU 使用率
- 其中 `container_cpu_usage_seconds_total` 代表 CPU 的计数器
- `container_spec_cpu_quota` 是容器的 CPU 配额，它的值是容器指定的 CPU 个数 \* 100000

```shell
sum(rate(container_cpu_usage_seconds_total{image!=""}[1m])) by (container, pod)  / (sum(container_spec_cpu_quota{image!=""}/100000) by (container, pod)) * 100
```

#### mem 属于不可压缩资源

- mem 属于**不可压缩资源**
- `Pod` 之间是**无法共享的，完全独占的**
- 一旦容器内存使用超过 limits，会导致 **`OOM`**，然后**重新调度**

#### mem OOM 判定依据

- `container_memory_working_set_bytes`是容器真实使用的内存量
- `kubelet` 通过比较 `container_memory_working_set_bytes` 和 `container_spec_memory_limit_bytes` 来决定 oom container
- 同时还有 `container_memory_usage_bytes` 用来表示容器使用内存，其中包含了很久没用的缓存，该值比 `container_memory_working_set_bytes` 要大
- 所以 mem 使用率可以使用下面的公式计算
  - `(container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100`

### CNI

Container Network Interface

### Authorization

鉴权执行链 unionAuthzHandler

#### 1. Node

一个专用鉴权组件，根据调度到 kubelet 上运行的 `Pod` 为 kubelet 授予权限。了解有关使用节点鉴权模式的更多信息，请参阅节点鉴权

#### 2. ABAC

基于属性的访问控制（ABAC）定义了一种访问控制范型，通过使用将属性组合 在一起的策略，将访问权限授予用户。策略可以使用任何类型的属性
  用户属性、资源属性、 对象，环境属性

#### 3. RBAC

基于角色的访问控制（RBAC）是一种基于企业内个人用户的角色来管理对计算机或网络资源的访问的方法。在此上下文中，权限是单个用户执行特定任务的能力，例如查看、创建或修改文件

被启用之后，RBAC（基于角色的访问控制）使用 `rbac.authorization.k8s.io` API 组来 驱动鉴权决策，从而允许管理员通过 Kubernetes API 动态配置权限策略

要启用 RBAC，请使用 `--authorization-mode = RBAC` 启动 API 服务器

##### Role 与 ClusterRole

- **Role** 命名空间级别
- **ClusterRole** 集群级别

##### RoleBinding 与 ClusterRoleBinding

- 角色绑定**将一个角色**中定义的各种**权限授予一个或者一组用户**
- 角色绑定包含了一组相关主体以及对被授予角色的引用
  - 相关主体 Subject
    - 用户 User
    - 用户组 Group
    - 服务账户 Service Account
- 在**命名空间**中可以通过 RoleBinding 对象授予权限，而**集群范围**的权限授予则通过 ClusterRoleBinding 对象完成。

#### 4. Webhook

WebHook 是一个 HTTP 回调：发生某些事情时调用的 HTTP POST； 通过 HTTP POST 进行简单的事件通知。实现 WebHook 的 Web 应用程序会在发生某些事情时 将消息发布到 URL

## Kubectl

**生成 pprof 火焰图:**

```bash
# 生成 .pprof
kubectl get pod -n ${ns} ${pod_name} --profile=xxx --profile-output=xxx

# 生成火焰图
go tool pprof -svg ${in.pprof} > {out.svg}
```

**部署资源：**

- `kubectl replace` 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象
- `kubectl apply`，则是执行了一个对原有 API 对象的 PATCH 操作

## etcd 的 tls 双向认证原理解析

![image](https://pic4.zhimg.com/v2-09903ee121fe3aead6b8c3f91e3c64c7_r.jpg)

### 双向认证的必备条件

- 私钥 client.key
- 个人认证证书 client.crt
- CA 根证书如 ca.crt
- CA中间证书（非所有情况下必需）
- 有了以上必备东西，当客户端验证服务器身份后，服务器才能验证客户端身份。双方都有自己独立的 SSL 证书，而且这些证书必须是由受信任的第三方 CA 机构颁发的
