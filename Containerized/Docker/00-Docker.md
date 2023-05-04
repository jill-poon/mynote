# Docker

## 介绍

### 什么是 Docker

> Go 语言实现的云开源项目

解决了运行环境和配置问题软件容器，方便做持续集成并有助于整体发布的容器虚拟化技术

**基于 Linux：**

1. namespace
    - 容器隔离
2. CGroups
    - Controller Groups
    - 资源限制，如内存、CPU
3. UnionFS
    Image 和 Container 分层

### 虚拟化技术

在物理机上虚拟出另一套完整的操作系统，能够使应用程序，操作系统和硬件三者之间的逻辑不变

**缺点：**

1. 资源占用多
2. 冗余步骤多
3. 启动慢

### 容器虚拟化技术

1. 容器不是模拟一个完整的操作系统，而是对进程进行隔离
2. 容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟
3. 高效轻量
4. 统一环境

## 基本组成

### Client

### Docker Host

#### Containers

运行实例

#### Images

> 只读模板

- UnionFS
  - 分层
  - 轻量级
  - 高性能
  - 文件系统
  - 它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。Union 文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像
  - 一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录
- 镜像加载原理
  - Bootfs(boot file system)主要包含 Bootloader 和 Kernel, Bootloader 主要是引导加载 Kernel, Linux 刚启动时会加载 Bootfs 文件系统，在 Docker 镜像的最底层是 bootfs。这一层与我们典型的 Linux/Unix 系统是一样的，包含 Boot 加载器和内核。当 boot 加载完成之后整个内核就都在内存中了，此时内存的使用权已由 bootfs 转交给内核，此时系统也会卸载 bootfs
  - Rootfs (root file system) ，在 Bootfs 之上。包含的就是典型 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。Rootfs 就是各种不同的操作系统发行版，比如 Ubuntu，Centos 等等。
- 分层结构的特点
  - 最大的一个好处就是共享资源，有多个镜像都从相同的 base 镜像构建而来，那么宿主机只需在磁盘上保存一份 base 镜像，同时内存中也只需加载一份 base 镜像，就可以为所有容器服务了。而且镜像的每一层都可以被共享

#### Volume (数据卷)

1. 持久化
    - 数据的持久化
    - 完全独立于容器的生存周期
    - 卷中的更改可以直接生效
    - 更改不会包含在镜像的更新中
2. 容器间共享
    - 数据卷可在容器之间共享或重用数据

> Docker 会把主机上的目录直接映射到容器的指定目录下，实现数据持久化

#### Network

- Badge
  - Network Namespce
        1. 隔离的网络空间
        2. 有独自的网络栈信息
        3. 要实现两个 network namespace 的通信，我们需要实现到的技术是 veth pair：Virtual Ethernet Pair，是一个成对的端口  
- Host
  - 容器将共享主机的网络堆栈
- None
  - 不会为容器配置任何 IP

### Registry

存放镜像

## Swarm

### 特点

- 官方提供的一款集群管理工具
- Swarm 和 Kubernetes 比较类似，但是更加轻，具有的功能也较 kubernetes 更少一些
- 自带，兼容 Docker API

### 管理节点

- 管理节点处理集群管理任务
    1. 使用 Raft 实现
    2. Majority (N-1)/2
    3. Docker 建议一个群最多有七个管理器节点

### 工作节点

唯一目的是执行容器

- 三不
  1. 不参与 Raft 分布式状态
  2. 不做出调度决策
  3. 不为 swarm 模式 HTTP API 提供服务

### Overlay

- 在网络技术领域，指的是一种网络架构上叠加的虚拟化技术模式，其大体框架是对基础网络不进行大规模修改的条件下，实现应用在网络上的承载，并能与其它网络业务分离，并且以基于 IP 的基础网络技术为主。
- VXLAN（Virtual eXtensible LAN）技术是当前最为主流的 Overlay 标准。
