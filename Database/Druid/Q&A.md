# Druid

## 线上数据库连接池泄漏，该如何排查？

日志

```log
exception=org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.exceptions.PersistenceException:
### Error querying database. Cause:
org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC connection; nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 60000, active 10, maxActive 10
```

* **`active` = `maxActive`= 10** 值太少，实际需要连接超过上限，把 **`maxActive`** 值设置为更大 (100)，重启服务，再去模拟线上的一个业务场景，可再过一段时间之后，也满了 (100)，不是线程池数量的问题，单纯修改 `maxActive` 配置不能解决。
* 开始排查业务代码，是否一直占用连接没有释放。Druid 默认没有开启监控功能，要首先**开启监控功能**，spring bean 配置 `ServletRegistrationBean` (配置用户名密码)、`FilterRegistrationBean` (配置过滤规则)，重启服务，登录监控界面，发现打开连接次数高达300+次，关闭=打开，活跃连接数是**0**，系统没有执行任何操作，数据是正常的。模拟业务场景，**打开数=关闭+活跃**，证明有活跃连接没有被释放，当长时间超过最大值时，会获取不到新的连接。通过代码是很难发现哪里出现连接泄露的，所以通过权重设置，来强制回收数据库连接，来引发程序报错，Druid 提供了 abandon 策略，去强制关闭正在活跃的连接，然后在 Druid 监控台查看报错堆栈信息，根据信息定位到具体代码，可以根据 *包名*，*SQL*语句去定位
