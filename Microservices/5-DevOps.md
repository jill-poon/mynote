# DevOps

## ​介绍

DevOps，字面意思是 Development & Operations 的缩写，也就是开发&运维

* ['dev] + [ˈɒps]
* 敏捷开发
* 可以快速交付
* 部署更加稳定

## 软件开发流程

### Plan

* 开发团队根据客户的目标制定开发计划

### Code

* 根据 PLAN 开始编码过程，需要将不同版本的代码存储在一个库中
* Git
* GitLab

### Build

* 编码完成后，需要将代码构建并且运行。
* Docker
* Docker Compose

### Test

* 成功构建项目后，需要测试代码是否存在 BUG 或错误。

### Deploy

* 代码经过手动测试和自动化测试后，认定代码已经准备好部署并且交给运维团队。

### Operate

* 运维团队将代码部署到生产环境中。

### Monitor

* 项目部署上线后，需要持续的监控产品

### Integrate

* 然后将监控阶段收到的反馈发送回 PLAN 阶段，整体反复的流程就是就是 DevOps 的核心，即持续集成、持续部署。署。
* Jenkins

## Pipeline

### 1 Checkout

1. JDK
2. Maven
3. GIt / GitLab

### 2 Build

1. Maven

### 3 Test

1. SonarQube

### 4 Publish

1. Docker
2. Harbor
3. Kubernetes
    * Kuboard
