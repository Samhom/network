## HTTP1.0、HTTP1.1 和 HTTP2.0 的区别

### 一、HTTP 的基本优化

影响一个 HTTP 网络请求的因素主要有两个：带宽和延迟。

- **带宽：**如果说我们还停留在拨号上网的阶段，带宽可能会成为一个比较严重影响请求的问题，但是现在网络基础建设已经使得带宽得到极大的提升，我们不再会担心由带宽而影响网速，那么就只剩下延迟了。

- **延迟：**

- - 浏览器阻塞（HOL blocking）：浏览器会因为一些原因阻塞请求。浏览器对于同一个域名，同时只能有 4 个连接（这个根据浏览器内核不同可能会有所差异），超过浏览器最大连接数限制，后续请求就会被阻塞。
  - DNS 查询（DNS Lookup）：浏览器需要知道目标服务器的 IP 才能建立连接。将域名解析为 IP 的这个系统就是 DNS。这个通常可以利用 DNS 缓存结果来达到减少这个时间的目的。
  - 建立连接（Initial connection）：HTTP 是基于 TCP 协议的，浏览器最快也要在第三次握手时才能捎带 HTTP 请求报文，达到真正的建立连接，但是这些连接无法复用会导致每次请求都经历三次握手和慢启动。三次握手在高延迟的场景下影响较明显，慢启动则对文件类大请求影响较大。

### 二、HTTP1.0 和 HTTP1.1 的一些区别

当前 HTTP1.1 依然是当前使用最为广泛的 HTTP 协议。

二者主要区别主要体现在：

1. **缓存处理**：在 HTTP1.0 中主要使用 header 里的 If-Modified-Since, Expires 来做为缓存判断的标准， HTTP1.1 则引入了更多的缓存控制策略例如 Entity tag，If-Unmodified-Since, If-Match, If-None-Match 等更多可供选择的缓存头来控制缓存策略。
2. **带宽优化及网络连接的使用**：HTTP1.0 中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1 则在请求头引入了 range 头域，它允许只请求资源的某个部分，即返回码是 206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。
3. **错误通知的管理**：在 HTTP1.1 中新增了 24 个错误状态响应码，如 409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
4. **Host 头处理**：在 HTTP1.0 中认为每台服务器都绑定一个唯一的 IP 地址，因此，请求消息中的 URL 并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个 IP 地址。HTTP1.1 的请求消息和响应消息都应支持 Host 头域，且请求消息中如果没有 Host 头域会报告一个错误（400 Bad Request）。
5. **长连接**：HTTP 1.1 支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个 TCP 连接上可以传送多个 HTTP 请求和响应，减少了建立和关闭连接的消耗和延迟，在 HTTP1.1 中默认开启Connection： keep-alive，一定程度上弥补了 HTTP1.0 每次请求都要创建连接的缺点。

### 三、SPDY：HTTP1.x 的优化

​	2012 年 google 如一声惊雷提出了 SPDY 的方案，优化了 HTTP1.X 的请求延迟，解决了 HTTP1.X 的安全性，具体如下：

1. **低延迟**：针对HTTP高延迟的问题，SPDY优雅的采取了多路复用（multiplexing）。多路复用通过多个请求stream共享一个tcp连接的方式，解决了HOL blocking的问题，降低了延迟同时提高了带宽的利用率。
2. **请求优先级**（request prioritization）：多路复用带来一个新的问题是，在连接共享的基础之上有可能会导致关键请求被阻塞。SPDY允许给每个request设置优先级，这样重要的请求就会优先得到响应。比如浏览器加载首页，首页的html内容应该优先展示，之后才是各种静态资源文件，脚本文件等加载，这样可以保证用户能第一时间看到网页内容。
3. **header压缩：**前面提到HTTP1.x的header很多时候都是重复多余的。选择合适的压缩算法可以减小包的大小和数量。
4. **基于HTTPS的加密协议传输**：大大提高了传输数据的可靠性。
5. **服务端推送**（server push）：采用了SPDY的网页，例如我的网页有一个sytle.css的请求，在客户端收到sytle.css数据的同时，服务端会将sytle.js的文件推送给客户端，当客户端再次尝试获取sytle.js时就可以直接从缓存中获取到，不用再发请求了。SPDY构成图：

![img](http://mmbiz.qpic.cn/mmbiz_png/cmOLumrNib1cfBOtIMQ6JfSibJdd6QkQribjhshzcKo97UNNVIFgpOYZic95drsxo5TaiadPSSmcYhOI7GYAO99W6Sw/0?wx_fmt=png)

​	SPDY 位于 HTTP 之下，TCP 和 SSL 之上，这样可以轻松兼容老版本的 HTTP 协议(将 HTTP1.x 的内容封装成一种新的 frame 格式)，同时可以使用已有的 SSL 功能。

### 四、HTTP2.0 性能惊人

​	Akamai 公司建立的一个官方的演示，用以说明 HTTP/2 相比于之前的 HTTP/1.1 在性能上的大幅度提升。 同时请求 379 张图片，从Load time 的对比可以看出 HTTP/2 在速度上的优势。

![img](http://mmbiz.qpic.cn/mmbiz_png/cmOLumrNib1cfBOtIMQ6JfSibJdd6QkQribcBaVkwLTtI3YprKtxYXhyYq7N1N70iaiaUxHFzc05wxQlSFrz3ltzrgw/0?wx_fmt=png)

​	HTTP2.0 可以说是 SPDY 的升级版（其实原本也是基于 SPDY 设计的），但是，HTTP2.0 跟 SPDY 仍有不同的地方，如下：

1. HTTP2.0 支持明文 HTTP 传输，而 SPDY 强制使用 HTTPS
2. HTTP2.0 和 SPDY 消息头的压缩算法不同

### 五、HTTP2.0 和 HTTP1.X 相比的新特性

- **新的二进制格式**（Binary Format）：HTTP1.x 的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认 0 和 1 的组合。基于这种考虑 HTTP2.0 的协议解析决定采用二进制格式，实现方便且健壮。
- **多路复用**（MultiPlexing）：即连接共享，即每一个 request 都是是用作连接共享机制的。一个 request 对应一个 id，这样一个连接上可以有多个 request，每个连接的 request 可以随机的混杂在一起，接收方可以根据request 的 id 将 request 再归属到各自不同的服务端请求里面。
- **header压缩**：如上文中所言，对前面提到过 HTTP1.x 的 header 带有大量信息，而且每次都要重复发送，HTTP2.0 使用 encoder 来减少需要传输的 header 大小，通讯双方各自 cache 一份 header fields 表，既避免了重复 header 的传输，又减小了需要传输的大小。
- **服务端推送**（server push）：同 SPDY 一样，HTTP2.0 也具有 server push 功能。

### 六、HTTP2.0 的多路复用和 HTTP1.X 中的长连接复用有什么区别？

- HTTP/1.* 一次请求-响应，建立一个连接，用完关闭；每一个请求都要建立一个连接；
- HTTP/1.1 Pipeling 解决方式为，若干个请求排队串行化单线程处理，后面的请求等待前面请求的返回才能获得执行机会，一旦有某请求超时等，后续请求只能被阻塞，毫无办法，也就是人们常说的线头阻塞；
- HTTP/2 多个请求可同时在一个连接上并行执行。某个请求任务耗时严重，不会影响到其它连接的正常执行；
  具体如图：

![img](http://mmbiz.qpic.cn/mmbiz_png/cmOLumrNib1cfBOtIMQ6JfSibJdd6QkQriba5ygCTOOjIQH4wvoJS2iaFBseyEAUfvpJQThHmTjuGuaSspUo8xppiaA/0?wx_fmt=png)

### 七、服务器推送到底是什么？

​	服务端推送能把客户端所需要的资源伴随着 index.html 一起发送到客户端，省去了客户端重复请求的步骤。正因为没有发起请求，建立连接等操作，所以静态资源通过服务端推送的方式可以极大地提升速度。具体如下：

#### 普通的客户端请求过程：

![img](http://mmbiz.qpic.cn/mmbiz_png/cmOLumrNib1cfBOtIMQ6JfSibJdd6QkQribkZLpDyHlTbAGkEiazqLfjkTSfMgib2UlC0p3Yw0T3iaaHcvLjL22PZWPg/0?wx_fmt=png)

#### 服务端推送的过程：

![img](http://mmbiz.qpic.cn/mmbiz_png/cmOLumrNib1cfBOtIMQ6JfSibJdd6QkQribq79os9yK2JmEODZqRVBweS7uMP2WWz4Ij6Z1f9TuiaXANOozhwCWljw/0?wx_fmt=png)

### 八、为什么需要头部压缩？

​	假定一个页面有100个资源需要加载（这个数量对于今天的Web而言还是挺保守的）, 而每一次请求都有1kb的消息头（这同样也并不少见，因为Cookie和引用等东西的存在）, 则至少需要多消耗100kb来获取这些消息头。HTTP2.0可以维护一个字典，差量更新HTTP头部，大大降低因头部传输产生的流量。具体参考：HTTP/2 头部压缩技术介绍。

### 九、HTTP2.0 多路复用有多好？

​	HTTP 性能优化的关键并不在于高带宽，而是低延迟。TCP 连接会随着时间进行自我「调谐」，起初会限制连接的最大速度，如果数据成功传输，会随着时间的推移提高传输的速度。这种调谐则被称为 TCP 慢启动。由于这种原因，让原本就具有突发性和短时性的 HTTP 连接变的十分低效。
HTTP/2 通过让所有数据流共用同一个连接，可以更有效地使用 TCP 连接，让高带宽也能真正的服务于 HTTP 的性能提升。