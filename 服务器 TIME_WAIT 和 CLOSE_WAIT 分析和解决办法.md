## 服务器 TIME_WAIT 和 CLOSE_WAIT 分析和解决办法

#### 首先需要了解三次握手与四次挥手：

![img](https://camo.githubusercontent.com/2d9c5a0729b689438be2428eb9d268be3ff7a83e/68747470733a2f2f696d672d626c6f672e6373646e2e6e65742f32303138303731373230323532303533313f77617465726d61726b2f322f746578742f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c334678587a4d344f5455774d7a45322f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f3730)

![img](https://camo.githubusercontent.com/c7b40a0c936565f97453d121aab3d76e2975da0c/68747470733a2f2f696d672d626c6f672e6373646e2e6e65742f32303138303731373230343230323536333f77617465726d61726b2f322f746578742f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c334678587a4d344f5455774d7a45322f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f3730)

##### 查看TIME_WAIT和CLOSE_WAIT数的命令：

```shell
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print (a,S[a])}'
LAST_ACK 14
SYN_RECV 348
ESTABLISHED 70
FIN_WAIT1 229
FIN_WAIT2 30
CLOSING 33
TIME_WAIT 18122
```

##### 状态描述：

```shell
CLOSED：无连接是活动的或正在进行
LISTEN：服务器在等待进入呼叫
SYN_RECV：一个连接请求已经到达，等待确认
SYN_SENT：应用已经开始，打开一个连接
ESTABLISHED：正常数据传输状态
FIN_WAIT1：应用说它已经完成
FIN_WAIT2：另一边已同意释放
ITMED_WAIT：等待所有分组死掉
CLOSING：两边同时尝试关闭
TIME_WAIT：另一边已初始化一个释放
LAST_ACK：等待所有分组死掉
```

##### 它会显示例如下面的信息：

TIME_WAIT 、CLOSE_WAIT 、FIN_WAIT1 、ESTABLISHED 、SYN_RECV 、LAST_ACK
常用的三个状态是：ESTABLISHED表示正在通信 、TIME_WAIT表示主动关闭、CLOSE_WAIT表示被动关闭。

##### 服务器出现异常最长出现的状况是：

- 服务器保持了大量的 TIME_WAIT 状态
- 服务器保持了大量的 CLOSE_WAIT 状态

​	我们也都知道 Linux 系统中分给每个用户的文件句柄数是有限的，而 TIME_WAIT 和 CLOSE_WAIT 这两种状态如果一直被保持，那么意味着对应数目的通道(此处应理解为 socket，一般一个 socket 会占用服务器端一个端口，服务器端的端口最大数是 65535)一直被占用，一旦达到了上限，则新的请求就无法被处理，接着就是大量Too Many Open Files 异常，然后 tomcat、nginx、apache 崩溃。

​	下面来讨论这两种状态的处理方法，网络上也有很多资料把这两种情况混为一谈，认为优化内核参数就可以解决，其实这是不恰当的。优化内核参数在一定程度上能解决 time_wait 过多的问题，但是应对 close_wait 还得从应用程序本身出发。

#### 1、服务器保持了大量的 time_wait 状态

​	这种情况比较常见，一般会出现在爬虫服务器和 web 服务器(如果没做内核参数优化的话)上，那么这种问题是怎么产生的呢？

​	time_wait 是主动关闭连接的一方保持的状态，对于爬虫服务器来说它自身就是客户端，在完成一个爬取任务后就会发起主动关闭连接，从而进入 time_wait 状态，然后保持这个状态 2MSL 时间之后，彻底关闭回收资源。这里为什么会保持资源 2MSL 时间呢？这也是 TCP/IP 设计者规定的。

​	TCP 要保证在所有可能的情况下使得所有的数据都能够被正确送达。当你关闭一个 socket 时，主动关闭一端的 socket 将进入TIME_WAIT 状 态，而被动关闭一方则转入 CLOSED 状态，这的确能够保证所有的数据都被传输。当一个 socket 关闭的时候，是通过两端四次握手完成的，当一端调用 close() 时，就说明本端没有数据要发送了。这好似看来在握手完成以后，socket 就都可以处于初始的 CLOSED 状态了，其实不然。原因是这样安排状态有两个问题， 首先，我们没有任何机制保证最后的一个 ACK 能够正常传输，第二，网络上仍然有可能有残余的数据包(wandering duplicates)，我们也必须能够正常处理。

​	TIMEWAIT 过多的话会占用内存，一个TIME_WAIT占用4k大小。

##### TIMEWAIT 就是为了解决这两个问题而生的：

- 假设最后的一个 ACK 丢失，那么被动关闭一方收不到这最后一个 ACK 则会重发 FIN。此时主动关闭一方必须保持一个有效的( time_wait 状态下维持)状态信息，以便可以重发 ACK。如果主动关闭的 socket 不维持这种状态而是进入 close 状态，那么主动关闭的一方在收到被动关闭方重新发送的 FIN 时则响应给被动方一个RST。被动方收到这个 RST 后会认为此次回话出错了。所以如果 TCP 想要完成必要的操作而终止双方的数据流传输，就必须完全正确的传输四次握手的四步，不能有任何的丢失。这就是为什么在 socket 在关闭后，任然处于 time_wait 状态的第一个原因。因为他要等待可能出现的错误(被动关闭端没有接收到最后一个 ACK，以便重发 ACK。
- 值得一说的是，基于 TCP 的 http 协议，一般(此处为什么说一般呢，因为当你在 keepalive 时间内主动关闭对服务器端的连接时，那么主动关闭端就是客户端，否则客户端就是被动关闭端。下面的爬虫例子就是这种情况)主动关闭 tcp 一端的是 server 端，这样 server 端就会进入 time_wait 状态，可想而知，对于访问量大的web 服务器，会存在大量的 time_wait 状态，假如 server 一秒钟接收 1000 个请求，那么就会积压240*1000=240000 个 time_wait 状态。(RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。)，维持这些状态给服务器端带来巨大的负担。当然现代操作系统都会用快速的查找算法来管理这些 TIME_WAIT，所以对于新的 TCP 连接请求，判断是否 hit 中一个 TIME_WAIT 不会太费时间，但是有这么多状态要维护总是不好。
- HTTP 协议 1.1 版本规定 default 行为是 keep-Alive，也就是会重用 tcp 连接传输多个 request/response。之所以这么做的主要原因是发现了我们上面说的这个问题。

TIME_WAIT 过多解决方案如下：

##### 1、编辑内核文件/etc/sysctl.conf，加入以下内容（然后执行 /sbin/sysctl -p 让参数生效.）：

```shell
net.ipv4.tcp_syncookies = 1 # 表示开启 SYN Cookies。当出现 SYN 等待队列溢出时，启用 cookies 来处理，可防范少量 SYN 攻击，默认为 0，表示关闭；
net.ipv4.tcp_tw_reuse = 1 # 表示开启重用。允许将 TIME-WAIT sockets 重新用于新的TCP连接，默认为0，表示关闭；
net.ipv4.tcp_tw_recycle = 1 # 表示开启TCP连接中 TIME-WAIT sockets 的快速回收，默认为0，表示关闭。
net.ipv4.tcp_fin_timeout # 修改系默认的 TIMEOUT 时间
```

简单来说，就是打开系统的 TIME_WAIT 重用和快速回收。

##### 2、客户端机器设置tcp_max_tw_buckets为一个很小的值：

​	表示系统同时保持 TIME_WAIT 套接字的最大数量，如果超过这个数字，TIME_WAIT 套接字将立刻被清除并打印警告信息。

​	默认为 180000，改为 5000。对于 Apache、Nginx 等服务器，上几行的参数可以很好地减少 TIME_WAIT 套接字数量。

#### 2、服务器保持了大量的 close_wait 状态

![img](https://img-blog.csdn.net/20171128155650107?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGVuZ2xpdTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

​	time_wait 问题可以通过调整内核参数和适当的设置 web 服务器的 keep-Alive 值来解决。因为 time_wait 是自己可控的，要么就是对方连接的异常，要么就是自己没有快速的回收资源，总之不是由于自己程序错误引起的。但是 close_wait 就不一样了，从上图中我们可以看到服务器保持大量的 close_wait 只有一种情况，那就是对方发送一个 FIN 后，程序自己这边没有进一步发送 ACK 以确认。换句话说就是在对方关闭连接后，程序里没有检测到，或者程序里本身就已经忘了这个时候需要关闭连接，于是这个资源就一直被程序占用着。这个时候快速的解决方法是：

- 关闭正在运行的程序，这个需要视业务情况而定。
- 尽快的修改程序里的 bug，然后测试提交到线上服务器。

#### 附：

​	服务器 A 是一台爬虫服务器，它使用简单的 HttpClient 去请求资源服务器 B 上面的 apache 获取文件资源，正常情况下，如果请求成功，那么在抓取完 资源后，服务器 A 会主动发出关闭连接的请求，这个时候就是主动关闭连接，服务器 A 的连接状态我们可以看到是 TIME_WAIT。如果一旦发生异常呢？假设请求的资源服务器 B 上并不存在，那么这个时候就会由服务器 B 发出关闭连接的请求，服务器 A 就是被动的关闭了连接，如果服务器 A 被动关闭连接之后程序员忘了让 HttpClient 释放连接，那就会造成 CLOSE_WAIT 的状态了。