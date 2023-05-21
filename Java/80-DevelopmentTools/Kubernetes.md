# Kubernetes 上的 Java 应用程序的最佳实践

在本文中，您将了解在 Kubernetes 上运行 Java 应用程序的最佳实践。这些建议中的大多数也适用于其他语言。但是，我正在考虑 Java 特征范围内的所有规则，并展示了可用于基于 JVM 的应用程序的解决方案和工具。其中一些 Kubernetes 建议在使用最流行的 Java 框架（如 Spring Boot 或 Quarkus）时被设计强制实施。

## 不要将 CPU 限制设置得太低

我们是否应该在 Kubernetes 上为 Java 应用程序设置限制？答案似乎是显而易见的。有许多工具可以验证您的 Kubernetes YAML 清单，如果您不设置 CPU 或内存限制，它们肯定会打印警告。但是，社区中对此有一些“热议”。这是一篇有趣的[文章](https://home.robusta.dev/blog/stop-using-cpu-limits)，不建议设置任何 CPU 限制。这是[另一篇文章](https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61)，作为前一篇文章的对比。他们考虑 CPU 限制，但我们也可以开始类似的内存限制讨论。特别是在 Java 应用程序的🙂上下文中

但是，对于内存管理，主张似乎大不相同。让我们[阅读另一篇文章](https://home.robusta.dev/blog/kubernetes-memory-limit) - 这次是关于内存限制和请求的。简而言之，它**建议始终设置内存限制。此外，限制应与请求相同**。在 Java 应用程序的上下文中，同样重要的是，我们可以使用 JVM 参数（如 `-Xmx`，`-XX:MaxMetaspaceSize` 或 `-XX:ReservedCodeCacheSize`）来限制内存。无论如何，从 Kubernetes 的角度来看，Pod 接收它请求的资源。限制与它无关。

这一切都让我想到了今天的第一个建议——不要把你的限制定得太低。即使您设置了 CPU 限制，也不会影响您的应用。例如，您可能知道，即使您的 Java 应用程序在正常工作中不消耗太多 CPU，它也需要大量 CPU 才能快速启动。对于我在 Kubernetes 上连接 MongoDB 的简单 Spring Boot [应用程序](https://github.com/piomin/sample-spring-boot-on-kubernetes.git)，无限制甚至 0.5 核心之间的差异是显着的。通常它会在 10 秒以下开始：

![kubernetes-java-startup](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-09-at-16.22.25.png?resize=696%2C58&ssl=1)

将 CPU 限制设置为 500 毫核时，它将启动 ~30 秒：

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-09-at-16.23.20.png?resize=696%2C56&ssl=1)

当然，我们可以找到一些例子。但我们也将在下一节中讨论它们。

## 首先考虑内存使用情况

让我们只关注内存限制。如果你在 Kubernetes 上运行一个 Java 应用程序，你有两个级别来限制最大使用量：容器和 JVM。但是，如果您没有为 JVM 指定任何设置，也有一些默认值。JVM 将其最大堆大小设置为可用 RAM 的大约 25%，以防您不设置该参数`-Xmx`。此值根据容器内可见的内存进行计数。一旦你没有在容器级别设置限制，JVM将看到节点的整个内存。

**在 Kubernetes 上运行应用程序之前，您至少应该测量它在预期负载下消耗的内存量。** 幸运的是，有一些工具可以优化容器中运行的 Java 应用程序的内存配置。例如，Paketo Buildpacks 带有一个内置的内存计算器。它使用公式 **`Heap = Total Container Memory - Non-Heap - Headroom`** 计算 `-Xmx` JVM 标志。另一方面，非堆值使用以下公式计算：**`Non-Heap = Direct Memory + Metaspace + Reserved Code Cache + (Thread Stack * Thread Count)`**。

Paketo Buildpacks 目前是构建 Spring Boot 应用程序的默认选项（使用命令 `mvn spring-boot:build-image`）。让我们为我们的示例应用尝试一下。假设我们将内存限制设置为512M，它将在130M 的级别上计算 `-Xmx`。

![kubernetes-java-memory](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-10-at-00.25.54.png?resize=696%2C60&ssl=1)

我的应用程序可以吗？我至少应该执行一些负载测试，以验证我的应用程序在繁忙的流量下的性能。但再说一遍——不要把限制定得太低。例如，对于1024M 限制，`-Xmx` 等于650M。

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-10-at-01.06.20.png?resize=696%2C58&ssl=1)

如您所见，我们使用 JVM 参数处理内存使用情况。它可以阻止我们在第一部分中提到的文章中描述的 OOM 杀戮。因此，**将请求设置为与限制相同的级别没有多大意义**。我**建议将其设置为比正常使用量高一点——假设多 20%**。

## 适当的 Liveness 和 Readiness 探测器

### 介绍

了解 Kubernetes 中 Liveness 和 Readiness 探测之间的区别至关重要。如果不仔细实施这两个探测器，它们可能会降低服务的整体操作，例如导致不必要的重新启动。第三种类型的探针，启动探针，是 Kubernetes 中相对较新的功能。它允许我们避免在Liveness或Readiness探测上设置 `initialDelaySecond` ，因此，如果您的应用启动需要大量时间，则特别有用。有关 Kubernetes 探针的一般和最佳实践的更多详细信息，我可以推荐那篇非常有趣的[文章](https://dnastacio.medium.com/the-art-and-science-of-probing-a-kubernetes-container-db1f16539080)。

**Liveness 探测器用于决定是否重新启动容器。** 如果应用程序由于任何原因不可用，重新启动容器有时是有意义的。另一方面，**Readiness 探测用于确定容器是否可以处理传入流量。** 如果某个 Pod 已被识别为未就绪，则会将其从负载平衡中移除。Readiness 探测失败不会导致容器重新启动。Web 应用程序最典型的 Liveness 或 Readiness 探测是通过 HTTP 终结点实现的。

由于 Liveness 探测的后续故障会导致 Pod 重新启动，因此它**不应检查应用集成的可用性**。此类内容应由 Readiness 探测进行验证。

### 配置详细信息

好消息是，最流行的 Java 框架，如 Spring Boot 或 Quarkus，提供了两种 Kubernetes 探测器的自动配置实现。他们遵循最佳实践，因此我们通常不必采取基础知识。但是，在 Spring Boot 中，除了 Actuator 模块之外，您还需要使用以下属性启用它们：

```yaml
management:
  endpoint: 
    health:
      probes:
        enabled: true
```

**由于 Spring Boot Actuator 提供了多个端点（例如 metrics, traces），因此最好将其公开在与默认端口不同的端口上（通常 `8080`）。** 当然，同样的规则也适用于其他流行的 Java 框架。另一方面，**一个好的做法是检查主应用端口，尤其是在 Readiness 中**。由于它定义了我们的应用程序是否准备好处理传入的请求，因此它也**应该侦听主端口**。它看起来与 Liveness 正好相反。**如果假设整个工作线程池都很忙，我不想重新启动我的应用程序。我只是不想在一段时间内收到传入的流量。**

我们还可以自定义 Kubernetes 探针的其他方面。假设我们的应用连接到外部系统，但我们未在 Readiness 中验证该集成。它并不重要，也不会对我们的运营状态产生直接影响。这是一个配置，它允许我们在探测中仅包含选定的集成集，并在主服务器端口上公开就绪情况。

```yaml
spring:
  application:
    name: sample-spring-boot-on-kubernetes
  data:
    mongodb:
      host: ${MONGO_URL}
      port: 27017
      username: ${MONGO_USERNAME}
      password: ${MONGO_PASSWORD}
      database: ${MONGO_DATABASE}
      authentication-database: admin

management:
  endpoint.health:
    show-details: always
    group:
      readiness:
        include: mongo # (1)
        additional-path: server:/readiness # (2)
    probes:
      enabled: true
  server:
    port: 8081
```

如果没有数据库、消息代理或其他应用程序等外部解决方案，我们的应用程序几乎无法存在。配置就绪探测时，我们应该仔细考虑与该系统的连接设置。首先，您应该考虑外部服务不可用的情况。你将如何处理它？我**建议将这些超时减少到较低的值**，如下所示。

```yaml
spring:
  application:
    name: sample-spring-kotlin-microservice
  datasource:
    url: jdbc:postgresql://postgres:5432/postgres
    username: postgres
    password: postgres123
    hikari:
      connection-timeout: 2000
      initialization-fail-timeout: 0
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
  rabbitmq:
    host: rabbitmq
    port: 5672
    connection-timeout: 2000
```

## 选择正确的 JDK

如果您已经使用 Dockerfile 构建了映像，则可能使用的是 Docker Hub 中的官方 OpenJDK 基础映像。但是，目前，图像[网站上的](https://hub.docker.com/_/openjdk)公告说它已正式弃用，所有用户都应该找到合适的替代品。我想这可能很令人困惑，所以你会[在这里](https://github.com/docker-library/openjdk/issues/505)找到原因的详细解释。

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-11-at-18.07.47.png?resize=671%2C434&ssl=1)

好吧，让我们考虑一下我们应该选择哪种选择。不同的供应商提供多种替代品。如果您正在寻找它们之间的详细比较，则应转到以下[站点](https://whichjdk.com/)。它建议在17版本中使用 Eclipse Temurin。

另一方面，最流行的映像构建工具（如 Jib 或云原生构建包）会自动为您选择供应商。默认情况下，Jib 使用 Eclipse Temurin，而 Paketo Buildpacks 使用 Bellsoft Liberica 实现。当然，您可以轻松覆盖这些设置。例如，如果您在与 JDK 提供商匹配的环境中运行应用程序，例如 AWS 和 Amazon Corretto，我认为这可能是有意义的。

假设我们使用 Paketo Buildpacks 和 Skaffold 在 Kubernetes 上部署 Java 应用程序。为了将默认的 Bellsoft Liberica 构建包替换为另一个构建包，我们只需要在构建包部分中逐字设置它。下面是一个利用 Amazon Corretto 构建包的示例。

```yaml
apiVersion: skaffold/v2beta22
kind: Config
metadata:
  name: sample-spring-boot-on-kubernetes
build:
  artifacts:
    - image: piomin/sample-spring-boot-on-kubernetes
      buildpacks:
        builder: paketobuildpacks/builder:base
        buildpacks:
          - paketo-buildpacks/amazon-corretto
          - paketo-buildpacks/java
        env:
          - BP_JVM_VERSION=17
```

我们还可以轻松地使用不同的 JDK 供应商测试应用程序的性能。如果您正在寻找此类比较的示例，可以阅读我描述此类测试和结果的文章。我测量了 Spring Boot 3应用程序的不同 JDK 性能，该应用程序使用几个可用的 Paketo Java Buildpack 与 Mongo 数据库交互。

## 考虑迁移到本机编译

原生编译是 Java 世界中真正的“游戏规则改变者”。但我敢打赌，你们中没有多少人使用它——尤其是在生产方面。当然，在将现有应用程序迁移到本机编译中时，存在（并且仍然存在）许多挑战。GraalVM 在构建时执行的静态代码分析可能会导致错误，例如 `ClassNotFound`、或 `MethodNotFound` 。为了克服这些挑战，我们需要提供一些提示，让 GraalVM 了解代码的动态元素。这些提示的数量通常取决于库的数量和应用程序中使用的语言功能的一般数量。

像 Quarkus 或 Micronaut 这样的 Java 框架试图通过设计来解决与本机编译相关的挑战。例如，他们尽可能避免使用反射。Spring Boot 还通过 Spring Native 项目改进了原生编译支持。因此，我在这方面的建议是，如果您要创建一个新应用程序，请以准备本机编译的方式准备它。例如，使用 Quarkus，您可以简单地生成一个 Maven 配置，其中包含用于构建本机可执行文件的专用配置文件。

```xml
<profiles>
  <profile>
    <id>native</id>
    <activation>
      <property>
        <name>native</name>
      </property>
    </activation>
    <properties>
      <skipITs>false</skipITs>
      <quarkus.package.type>native</quarkus.package.type>
    </properties>
  </profile>
</profiles>
```

添加后，您可以使用以下命令进行本机构建：

```bash
$ mvn clean package -Pnative
```

然后，您可以分析构建过程中是否存在任何问题。即使现在不在生产环境中运行本机应用（例如，组织不批准它），也应将 GraalVM 编译作为接受管道中的一个步骤。您可以使用最流行的框架轻松为您的应用构建 Java 本机映像。例如，使用 Spring Boot，您只需要在 Maven 中提供以下配置`pom.xml`，如下所示：

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>build-info</goal>
        <goal>build-image</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <image>
      <builder>paketobuildpacks/builder:tiny</builder>
      <env>
        <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
        <BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
          --allow-incomplete-classpath
        </BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
      </env>
    </image>
  </configuration>
</plugin>
```

## 正确配置日志记录

日志记录可能不是您在编写 Java 应用程序时首先考虑的事情。但是，在全球范围内，它变得非常重要，因为我们需要能够快速收集、存储数据并最终搜索和呈现特定条目。最佳做法是将应用程序日志写入标准输出 （_stdout_） 和标准错误 （_stderr_） 流。

Fluentd 是一种流行的开源日志聚合器，允许您从 Kubernetes 集群收集日志，处理它们，然后将它们发送到您选择的数据存储后端。它与 Kubernetes 部署无缝集成。Fluentd 尝试将数据构建为 JSON，以统一不同源和目标之间的日志记录。假设如此，最好的方法可能是以这种格式准备日志。使用 JSON 格式，我们还可以轻松包含用于标记日志的其他字段，然后使用各种条件在可视化工具中轻松搜索它们。

为了将我们的日志格式化为 Fluentd 可读的 JSON，我们可能会在 Maven 依赖项中包含 Logstash Logback Encoder 库。

```xml
<dependency>
   <groupId>net.logstash.logback</groupId>
   <artifactId>logstash-logback-encoder</artifactId>
   <version>7.2</version>
</dependency>
```

然后我们只需要在文件中为我们的 Spring 启动应用程序设置一个默认的控制台日志附加器`logback-spring.xml`。

```xml
<configuration>
    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    <logger name="jsonLogger" additivity="false" level="DEBUG">
        <appender-ref ref="consoleAppender"/>
    </logger>
    <root level="INFO">
        <appender-ref ref="consoleAppender"/>
    </root>
</configuration>
```

我们是否应该避免使用额外的日志追加器，而只将日志打印到标准输出？根据我的经验，答案是否定的。您仍然可以使用其他机制来发送日志。特别是如果您使用多个工具来收集组织中的日志 - 例如 Kubernetes 上的内部堆栈和外部的全局堆栈。就个人而言，我正在使用一种工具来帮助我解决性能问题，例如消息代理作为代理。在 Spring Boot 中，我们可以轻松地使用 RabbitMQ。只需包括以下启动器：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

然后，您需要在：`logback-spring.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

  <springProperty name="destination" source="app.amqp.url" />

  <appender name="AMQP"
		class="org.springframework.amqp.rabbit.logback.AmqpAppender">
    <layout>
      <pattern>
{
  "time": "%date{ISO8601}",
  "thread": "%thread",
  "level": "%level",
  "class": "%logger{36}",
  "message": "%message"
}
      </pattern>
    </layout>

    <addresses>${destination}</addresses>	
    <applicationId>api-service</applicationId>
    <routingKeyPattern>logs</routingKeyPattern>
    <declareExchange>true</declareExchange>
    <exchangeName>ex_logstash</exchangeName>

  </appender>

  <root level="INFO">
    <appender-ref ref="AMQP" />
  </root>

</configuration>
```

## 创建集成测试

好的，我知道 - 它与 Kubernetes 没有直接关系。但是由于我们使用 Kubernetes 来管理和编排容器，我们也应该在容器上运行集成测试。幸运的是，使用 Java 框架，我们可以大大简化该过程。例如，Quarkus 允许我们使用 `@QuarkusIntegrationTest`。这是一个非常强大的解决方案，与 Quarkus 容器构建功能相结合。我们可以针对包含应用的已构建映像运行测试。首先，让我们包含夸库斯动臂模块：

```xml
<dependency>
   <groupId>io.quarkus</groupId>
   <artifactId>quarkus-container-image-jib</artifactId>
</dependency>
```

然后，我们必须通过在 application.properties 文件中将 quarkus.container-image.build 属性设置为 true 来启用容器构建。在测试类中，我们可以使用 `@TestHTTPResource` 和 `@TestHTTPEndpoint` 注释来注入测试服务器 URL。然后，我们将创建一个客户端，并调用 `RestClientBuilder` 在容器上启动的服务。测试类的名称并非偶然。为了自动检测为集成测试，它具有 `IT` 后缀。

```java
@QuarkusIntegrationTest
public class EmployeeControllerIT {

    @TestHTTPEndpoint(EmployeeController.class)
    @TestHTTPResource
    URL url;

    @Test
    void add() {
        EmployeeService service = RestClientBuilder.newBuilder()
                .baseUrl(url)
                .build(EmployeeService.class);
        Employee employee = new Employee(1L, 1L, "Josh Stevens", 
                                         23, "Developer");
        employee = service.add(employee);
        assertNotNull(employee.getId());
    }

    @Test
    public void findAll() {
        EmployeeService service = RestClientBuilder.newBuilder()
                .baseUrl(url)
                .build(EmployeeService.class);
        Set<Employee> employees = service.findAll();
        assertTrue(employees.size() >= 3);
    }

    @Test
    public void findById() {
        EmployeeService service = RestClientBuilder.newBuilder()
                .baseUrl(url)
                .build(EmployeeService.class);
        Employee employee = service.findById(1L);
        assertNotNull(employee.getId());
    }
}
```

您可以在我之前关于 Quarkus 高级测试[的文章中](https://piotrminkowski.com/2023/02/08/advanced-testing-with-quarkus/)找到有关该过程的更多详细信息。最终效果如下图所示。当我们使用 mvn clean verify 命令在构建期间运行测试时，我们的测试是在构建容器映像后执行的。

![kubernetes-java-integration-tests](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2023/02/Screenshot-2023-02-08-at-10.30.43.png?resize=696%2C321&ssl=1)

Quarkus 功能基于 Testcontainers 框架。我们还可以将 Testcontainers 与 Spring Boot 一起使用。以下是 Spring REST 应用程序及其与 PostgreSQL 数据库集成的示例测试。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class PersonControllerTests {

    @Autowired
    TestRestTemplate restTemplate;

    @Container
    static PostgreSQLContainer<?> postgres = 
       new PostgreSQLContainer<>("postgres:15.1")
            .withExposedPorts(5432);

    @DynamicPropertySource
    static void registerMySQLProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    @Order(1)
    void add() {
        Person person = Instancio.of(Person.class)
                .ignore(Select.field("id"))
                .create();
        person = restTemplate.postForObject("/persons", person, Person.class);
        Assertions.assertNotNull(person);
        Assertions.assertNotNull(person.getId());
    }

    @Test
    @Order(2)
    void updateAndGet() {
        final Integer id = 1;
        Person person = Instancio.of(Person.class)
                .set(Select.field("id"), id)
                .create();
        restTemplate.put("/persons", person);
        Person updated = restTemplate.getForObject("/persons/{id}", Person.class, id);
        Assertions.assertNotNull(updated);
        Assertions.assertNotNull(updated.getId());
        Assertions.assertEquals(id, updated.getId());
    }

}
```

## 结语

我希望这篇文章能帮助你在 Kubernetes 上运行 Java 应用程序时避免一些常见的陷阱。把它当作我在类似文章中发现的其他人的建议和我在这方面的私人经验的总结。也许你会发现其中一些规则是相当有争议的。请随时在评论中分享您的意见和反馈。这对我来说也很有价值。如果你喜欢这篇文章，我再次推荐阅读我博客中的另一篇文章——更专注于在 Kubernetes 上运行基于微服务的应用程序——[Kubernetes 上的微服务最佳实践](https://piotrminkowski.com/2020/03/10/best-practices-for-microservices-on-kubernetes/)。但它也包含几个有用的（我希望）建议。