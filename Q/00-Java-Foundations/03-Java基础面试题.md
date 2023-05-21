# Java基础面试题，JavaWeb专题

## 1.HTTP响应码有哪些

1、1xx（临时响应）
2、2xx（成功）
3、3xx（重定向）：表示要完成请求需要进一步操作
4、4xx（错误）：表示请求可能出错，妨碍了服务器的处理
5、5xx（服务器错误）：表示服务器在尝试处理请求时发生内部错误

举例：

200：成功，Web服务器成功处理了客户端的请求。
301：永久重定向，当客户端请求一个网址的时候，Web服务器会将当前请求重定向到另一个网址，搜索引擎会抓取重定向后网页的内容并且将旧的网址替换为重定向后的网址。
302：临时重定向，搜索引擎会抓取重定向后网页的内容而保留旧的网址，因为搜索引擎认为重定向后的网址是暂时的。
400：客户端请求错误，多为参数不合法导致Web服务器验参失败。
404：未找到，Web服务器找不到资源。
500：Web服务器错误，服务器处理客户端请求的时候发生错误。
503：服务不可用，服务器停机。
504：网关超时

## 2.Forward 和 Redirect 的区别？

1. 浏览器 URL 地址：Forward 是服务器内部的重定向，服务器内部请求某个 servlet，然后获取响应的内容，浏览器的 URL 地址是不会变化的；Redirect 是客户端请求服务器，然后服务器给客户端返回了一个302状态码和新的 location，客户端重新发起 HTTP 请求，服务器给客户端响应 location 对应的 URL 地址，浏览器的 URL 地址发生了变化。
2. 数据的共享：Forward是服务器内部的重定向，request在整个重定向过程中是不变的，request中的信息在servlet间是共享的。Redirect发起了两次HTTP请求分别使用不同的request。
3. 请求的次数：Forward只有一次请求；Redirect有两次请求。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/fba756c1b94e439ab2dec986650a7117.png)

## 3.Get 和 Post 请求的区别

用途：

* get请求用来从服务器获取资源
* post请求用来向服务器提交数据

表单的提交方式：

* get请求直接将表单数据以name1=value1&name2=value2的形式拼接到URL上（<http://www.baidu.com/action?name1=value1&name2=value2），多个参数参数值需要用&连接起来并且用?拼接到action后面；>
* post请求将表单数据放到请求头或者请求的消息体中。

传输数据的大小限制：

* get请求传输的数据受到URL长度的限制，而URL长度是由浏览器决定的；
* post请求传输数据的大小理论上来说是没有限制的。

参数的编码：

* get请求的参数会在地址栏明文显示，使用URL编码的文本格式传递参数；
* post请求使用二进制数据多重编码传递参数。

缓存处理：

* get请求可以被浏览器缓存被收藏为标签；
* post请求不会被缓存也不能被收藏为标签

## 4.介绍下OSI七层和TCP/IP四层的关系

为了更好地促进互联网的研究和发展，国际标准化组织ISO在1985 年指定了网络互联模型。OSI 参考模型（Open System Interconnect Reference [Model](https://so.csdn.net/so/search?q=Model&spm=1001.2101.3001.7020)），具有 7 层结构

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/8b05a5e5ca8f45f6adfe5a2473797a6e.png)

**应用层**：各种应用程序协议，比如HTTP、HTTPS、FTP、SOCKS安全套接字协议、DNS域名系统、GDP网关发现协议等等。
**表示层**：加密解密、转换翻译、压缩解压缩，比如LPP轻量级表示协议。
**会话层**：不同机器上的用户建立和管理会话，比如SSL安全套接字层协议、TLS传输层安全协议、RPC远程过程调用协议等等。

**传输层**：接受上一层的数据，在必要的时候对数据进行分割，并将这些数据交给网络层，保证这些数据段有效到达对端，比如TCP传输控制协议、UDP数据报协议。
**网络层**：控制子网的运行：逻辑编址、分组传输、路由选择，比如IP、IPV6、SLIP等等。
**数据链路层**：物理寻址，同时将原始比特流转变为逻辑传输路线，比如XTP压缩传输协议、PPTP点对点隧道协议等等。
**物理层**：机械、电子、定时接口通信信道上的原始比特流传输，比如IEEE802.2等等。

而且在消息通信的过程中具体的执行流程为：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/62a425220acc40b6b53cf2c34a4331a6.png)

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/933af2e7173942319f49900f5ce9ae49.png)

网络传输的数据其实会通过这七层协议来进行数据的封装和拆解

## 5.说说 TCP 和 UDP 的区别

1. TCP 面向连接（如打电话要先拨号建立连接）：UDP 是无连接的，即发送数据之前不需要建立连接。
2. TCP 提供可靠的服务。也就是说，通过 TCP 连接传送的数据，无差错，不丢失，不重复，且按序到达;UDP 尽最大努力交付，即不保证可靠交付。tcp 通过校验和，重传控制，序号标识，滑动窗口、确认应答实现可靠传输。如丢包时的重发控制，还可以对次序乱掉的分包进行顺序控制。
3. UDP 具有较好的实时性，工作效率比 TCP 高，适用于对高速传输和实时性有较高的通信或广播通信。
4. 每一条 TCP 连接只能是点到点的;UDP 支持一对一，一对多，多对一和多对多的交互通信
5. TCP 对系统资源要求较多，UDP 对系统资源要求较少。

## 6. 说下 HTTP 和 HTTPS 的区别

端口不同：HTTP 和 HTTPS 的连接方式不同没用的端口也不一样，HTTP 是80，HTTPS 用的是443
消耗资源：和 HTTP 相比，HTTPS 通信会因为加解密的处理消耗更多的 CPU 和内存资源。
开销：HTTPS 通信需要证书，这类证书通常需要向认证机构申请或者付费购买。

## 7. 说下 HTTP、TCP、Socket 的关系是什么？

* TCP/IP 代表传输控制协议/网际协议，指的是一系列协议族。
* HTTP 本身就是一个协议，是从 Web 服务器传输超文本到本地浏览器的传送协议。
* Socket 是 TCP/IP 网络的 API，其实就是一个门面模式，它把复杂的 TCP/IP 协议族隐藏在 Socket 接口后面。对用户来说，一组简单的接口就是全部，让 Socket 去组织数据，以符合指定的协议。

综上所述：

* 需要 IP 协议来连接网络
* TC P 是一种允许我们安全传输数据的机制，使用 TCP 协议来传输数据的 HTTP 是 Web 服务器和客户端使用的特殊协议。
* HTTP 基于 TCP 协议，所以可以使用 Socket 去建立一个 TCP 连接。

## 8. 说下HTTP的长链接和短连接的区别

HTTP协议的长连接和短连接，实质上是TCP协议的长连接和短连接。

**短连接**
在 HTTP/1.0中默认使用短链接,也就是说，浏览器和服务器每进行一次 HTTP 操作，就建立一次连接，但任务结束就中断连接。如果客户端访问的某个 HTML 或其他类型的 Web 资源，如 JavaScript 文件、图像文件、CSS 文件等。当浏览器每遇到这样一个 Web 资源，就会建立一个 HTTP 会话。

**长连接**
从 HTTP/1.1起，默认使用长连接，用以保持连接特性。在使用长连接的情况下，当一个网页打开完成后，客户端和服务器之间用于传输 HTTP 数据的 TCP 连接不会关闭。如果客户端再次访问这个服务器上的网页，会继续使用这一条已经建立的连接。Keep-Alive 不会永久保持连接，它有一个保持时间，可以在不同的服务器软件（如 Apache）中设定这个时间。

## 9.TCP原理

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/2c3e7415ef194f0a93fd91345ba9c0ac.png)

三次握手：

1. 第一次握手：客户端将标志位 syn 重置为1，随机产生 seq=a，并将数据包发送给服务端
2. 第二次握手：服务端收到 syn=1知道客户端请求连接，服务端将 syn 和 ACK 都重置为1，ack=a+1，随机产一个值 seq=b，并将数据包发送给客户端，服务端进入 syn_RCVD 状态。
3. 第三次握手：客户端收到确认后，检查 ack 是否为 a+1，ACK 是否为1，若正确将 ACK 重置为1，将 ack 改为 b+1，然后将数据包发送给服务端服务端检查 ack 与 ACK,若都正确，就建立连接，进入 ESTABLISHEN.

四次挥手：

1. 开始双方都处于连接状态
2. 客户端进程发出 FIN 报文，并停止发送数据，在报文中 FIN 结束标志为1，seq 为 a 连接状态下发送给服务器的最后一个字节的序号+1，报文发送结束后，客户端进入 FIN-WIT1状态。
3. 服务端收到报文，向客户端发送确认报文，ACK=1,seq 为 b 服务端给客户端发送的最后字节的序号+1，ack=a+1，发送后客户端进入 close-wait 状态，不再发送数据，但服务端发送数据客户端一九可以收到（称为半关闭状态）。
4. 客户端收到服务器的确认报文后，客户端进入 fin-wait2状态进行等待服务器发送第三次的挥手报文。
5. 服务端向 fin 报文 FIN=1ACK=1，seq=c（服务器向客户端发送最后一个字节序号+1），ack=b+1，发送结束后服务器进入 last-ack 状态等待最后的确认。
6. 客户端收到是释放报文后，向服务器发送确认报文进入 time-wait 状态，后进入 close
7. 服务端收到确认报文进入 close 状态。

## 10. Cookie 和 Session 的区别

cookie 是由 Web 服务器保存在用户浏览器上的文件（key-value 格式），可以包含用户相关的信息。客户端向服务器发起请求，就提取浏览器中的用户信息由 http 发送给服务器

session 是浏览器和服务器会话过程中，服务器会分配的一块储存空间给 session。服务器默认为客户浏览器的 cookie 中设置 sessionid，这个 sessionid 就和 cookie 对应，浏览器在向服务器请求过程中传输的 cookie 包含 sessionid，服务器根据传输 cookie 中的 sessionid 获取出会话中存储的信息，然后确定会话的身份信息。

1. Cookie 数据存放在客户端上，安全性较差，Session 数据放在服务器上，安全性相对更高
2. 单个 cookie 保存的数据不能超过4K，session 无此限制
3. session 一定时间内保存在服务器上，当访问增多，占用服务器性能，考虑到服务器性能方面，应当使用 cookie。

## 11.Tomcat 是什么？

Tomcat 服务器 Apache 软件基金会项目中的一个核心项目，是一个免费的开放源代码的 Web 应用服务器（Servlet 容器），属于轻量级应用服务器，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调试 JSP 程序的首选。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/cc3c3c6020d54b098e862692bada4302.png)

## 12. Tomcat 有几种部署方式

1. 利用 Tomcat 的自动部署：把 web 应用拷贝到 webapps 目录（生产环境不建议放在该目录中）。Tomcat 在启动时会加载目录下的应用，并将编译后的结果放入 work 目录下。
2. 使用 Manager App 控制台部署：在 tomcat 主页点击“Manager App”进入应用管理控制台，可以指定一个 web 应用的路径或 war 文件。
3. 修改 conf/server.xml 文件部署：在 server.xml 文件中，增加 Context 节点可以部署应用。
4. 增加自定义的 Web 部署文件：在 conf/Catalina/localhost/路径下增加 xyz.xml 文件，内容是 Context 节点，可以部署应用。

## 13. 什么是 Servlet

Servlet 是 JavaEE 规范的一种，主要是为了扩展 Java 作为 Web 服务的功能，统一接口。由其他内部厂商如 tomcat，jetty 内部实现 web 的功能。如一个 http 请求到来：容器将请求封装为 servlet 中的 HttpServletRequest 对象，调用 init()，service()等方法输出 response，由容器包装为 httpresponse 返回给客户端的过程。

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/f1f331019c8b41519bd49a588d413ffc.png)

## 14. 什么是 Servlet 规范?

* 从 Jar 包上来说，Servlet 规范就是两个 Jar 文件。servlet-api.jar 和 jsp-api.jar，Jsp 也是一种 Servlet。
* 从 package 上来说，就是 javax.servlet 和 javax.servlet.http 两个包。
* 从接口来说，就是规范了 Servlet 接口、Filter 接口、Listener 接口、ServletRequest 接口、ServletResponse 接口等。类图如下：

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/5605477c3db841e7824a0432e4a6c2a9.png)

## 15.为什么我们将 tomcat 称为 Web 容器或者 Servlet 容器？

我们用一张图来表示他们之间的关系:

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/5e137f734ac540ed9aaee12704166ec0.png)

简单的理解：启动一个 ServerSocket，监听8080端口。Servlet 容器用来装我们开发的 Servlet。

## 16.Servlet 的生命周期

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/b6757fdbc638457daebd04fd83a9d0f7.png)

## 17. jsp 和 Servlet 的区别

* 本质都是servlet
* servlet侧重于逻辑处理
* jsp侧重于视图显示

## 18. 九大内置对象

1. page 页面对象
2. config 配置对象
3. request 请求对象
4. response 响应对象
5. session 会话对象
6. application 全局对象
7. out 输出对象
8. pageContext 页面上下文对象
9. exception 异常对象

## 19. JSP的四大作用域

page：

```txt
只在当前页面有效
```

request：

```txt
它在当前请求中有效
```

session：

```txt
它在当前会话中有效
```

application：

```txt
他在所有的应用程序中都有效
```

注意：当4个作用域对象都有相同的name属性时,默认按照最小的顺序查找

## 20. **GenericServlet 和 HttpServlet 有什么区别？**

GenericServlet 为抽象类，定义了一个通用的、独立于底层协议的 servlet，实现了 Servlet 和 ServletConfig 接口，ServletConfig 接口定义了在 Servlet 初始化的过程中由 Servlet 容器传递给 Servlet 得配置信息对象。OK，这个类可能我们不是那么熟悉，但是他的子类相信大家都知道，也就是 HttpServlet，HttpServlet 继承自抽象类 GenericServlet 具有其所有的特性并拓展了一些其他的方法,如 doGet、doPost 等

![image.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/fyfile/1462/1675489425009/12c9c504a75243bba4aec073404bdc5e.png)
