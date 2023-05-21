# SkyWalking

  

## 什么是 SkyWalking

[SkyWalking](https://github.com/apache/skywalking) 用于分布式系统的 APM (应用程序性能监视工具)，特别是为微服务、云本地和基于容器(Kubernetes)架构设计的。

## APM

APM 全称 Application Performance Management 应用性能管理，目的是通过各种探针采集数据，收集关键指标，同时搭配数据呈现以实现对应用程序性能管理和故障管理的系统化解决方案。

Zabbix、Premetheus、open-falcon 等监控系统主要***关注服务器硬件指标***与***系统服务运行状态***等，而 APM 系统则更重视**程序内部执行过程**指标和**服务之间链路调用**情况的监控，APM 更有利于深入代码找到请求响应“慢”的根本问题，与 Zabbix 之类的监控是互补关系目前市面上开源的 APM 系统主要有 CAT、Zipkin、Pinpoint、SkyWalking，大都是参考 Google 的 `Dapper` 实现的。

## 链路追踪工具对比

链路追踪工具一般要有如下功能：

* 心跳检测（确定应用是否还在运行）
* 记录请求的执行流程、执行时间
* 资源监控（CPU、内存、带宽、磁盘）
* 告警功能（监控执行时间、成功率等通过邮件、钉钉、短信、微信等进行通知）
* 可视化页面

| 对比项               | Zipkin                                  | Pinpoint             | SkyWalking           | Cat                              |
| -------------------- | --------------------------------------- | -------------------- | -------------------- | -------------------------------- |
| 实现方式             | 拦截请求，发送(Http,MQ)数据到Zipkin服务 | Java探针，字节码增强 | Java探针，字节码增强 | 代码埋点(拦截器，注解，过滤器等) |
| 接入方式             | 基于linkerd或者sleuth方式               | javaagent字节码      | javaagent字节码      | 代码侵入                         |
| agent到collector协议 | http,MQ                                 | thrift               | gRPC                 | http/tcp                         |
| OpenTracing          | 支持                                    | 不支持               | 支持                 | 不支持                           |
| 颗粒度               | 接口级                                  | 方法级               | 方法级               | 代码级                           |
| 全局调用统计         | 不支持                                  | 支持                 | 支持                 | 支持                             |
| traceid查询          | 支持                                    | 不支持               | 支持                 | 不支持                           |
| 报警                 | 不支持                                  | 支持                 | 支持                 | 支持                             |
| JVM监控              | 不支持                                  | 不支持               | 支持                 | 支持                             |
| UI功能               | 支持                                    | 支持                 | 支持                 | 支持                             |
| 数据存储             | ES、MySQL 等                            | HBase                | ES/H2/MySQL          | MySQL/HDFS                       |

## 自定义 SkyWalking 链路

默认情况下 SkyWalking 是没有记录我们的业务方法的，如果需要添加业务方法的链路监控我们就需要添加如下的依赖，然后在业务方法上添加 `@Trace` 注解。

```xml

<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>x.y.z</version>
</dependency>

```

可以通过 `@Tags` 和 `@Tag` 来解决方法的详情中没有返回信息和参数的问题

```java

@Trace
@Tags({
    @Tag(key = "method", value = "returnedValue"),
    @Tag(key = "param", value = "arg[0]")
})
```
  
## 集成日志框架

将微服务的日志框架去集成 SkyWalking，我们希望在我们微服务中日志中，能够记录当前调用链路的 id，然后我们再根据这个 id 去 SkyWalking 的前端界面中进行搜索找到对应的调用链路记录。

因为springboot默认实现的日志框架是logback，这里也就拿logback举例

```xml
<!-- skywalking 日志记录  -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>8.5.0</version>
</dependency>
```

在项目中 `resources`目录下创建 `logback-spring.xml`文件

```xml

<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level logger_name:%logger{36} - [%tid] - message:%msg%n</pattern>
            </layout>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="console" />
    </root>
</configuration>
```

在 SkyWalking UI 的日志菜单中显示日志信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--  控制台日志输出的格式中添加tid  -->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level logger_name:%logger{36} - [%tid] - message:%msg%n</pattern>
            </layout>
        </encoder>
    </appender>
    <!-- skywalking grpc 日志收集 8.4.0版本开始支持 -->
    <appender name="grpc-log" class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} -%msg%n</Pattern>
            </layout>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="console" />
        <appender-ref ref="grpc-log" />
    </root>
</configuration>
```

```properties
plugin.toolkit.log.grpc.reporter.server_host=${SW_GRPC_LOG_SERVER_HOST:127.0.0.1}
plugin.toolkit.log.grpc.reporter.server_port=${SW_GRPC_LOG_SERVER_PORT:11800}
plugin.toolkit.log.grpc.reporter.max_message_size=${SW_GRPC_LOG_MAX_MESSAGE_SIZE:10485760}
plugin.toolkit.log.grpc.reporter.upstream_timeout=${SW_GRPC_LOG_GRPC_UPSTREAM_TIMEOUT:30}
```