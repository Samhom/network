## HTTP缓存机制

​	HTTP 中具有缓存功能的是浏览器缓存。HTTP缓存作为web性能优化的重要手段。

### 1、缓存方案

​	与缓存相关的规则信息就包含在HTTP报文的header中。HTTP报文header部分的例子如下：

![img](https://user-gold-cdn.xitu.io/2017/11/29/16007e57ca9f8f86?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

​	那么缓存机制的运行原理是怎样的，而服务器又是如何判断缓存是否失效呢？接着看下边的分析。

### 2、缓存规则

​	HTTP 缓存属于客户端缓存，可认为浏览器存在一个缓存数据库，用于储存一些不经常变化的静态文件（图片、css、js 等）。缓存分为**强制缓存**和**协商缓存**。

#### 强制缓存

​	当缓存数据库中已有所请求的数据时。客户端直接从缓存数据库中获取数据。当缓存数据库中没有所请求的数据时，客户端的才会从服务端获取数据。

![img](https://user-gold-cdn.xitu.io/2017/11/29/16007be6f64ff7f7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

​	对于强制缓存，服务器响应的header中会用两个字段来表明——**Expires** 和 **Cache-Control**。

##### Expires

​	Exprires 的值为服务端返回的数据到期时间。当再次请求时的请求时间小于返回的此时间，则直接使用缓存数据。但由于服务端时间和客户端时间可能有误差，这也将导致缓存命中的误差，另一方面，Expires是HTTP1.0的产物，故现在大多数使用Cache-Control替代。

##### Cache-Control

Cache-Control 有很多属性，不同的属性代表的意义也不同：

- private：客户端可以缓存 
- public：客户端和代理服务器都可以缓存 
- max-age=t：缓存内容将在 t 秒后失效 
- no-cache：需要使用协商缓存来验证缓存数据 
- no-store：所有内容都不会缓存。



#### 协商缓存

​	又称对比缓存，客户端会先从缓存数据库中获取到一个缓存数据的标识，得到标识后请求服务端验证是否失效（新鲜），如果没有失效服务端会返回304，此时客户端直接从缓存中获取所请求的数据，如果标识失效，服务端会返回更新后的数据。

![img](https://user-gold-cdn.xitu.io/2017/11/29/16007d1c835d5461?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

​	协商缓存需要进行对比判断是否可以使用缓存。浏览器第一次请求数据时，服务器会将缓存标识与数据一起响应给客户端，客户端将它们备份至缓存中。再次请求时，客户端会将缓存中的标识发送给服务器，服务器根据此标识判断。若未失效，返回 304 状态码，浏览器拿到此状态码就可以直接使用缓存数据了。 对于协商缓存来说，缓存标识我们需要着重理解一下，下面我们将着重介绍它的两种缓存方案。

#### Last-Modified

Last-Modified： 服务器在响应请求时，会告诉浏览器资源的最后修改时间。

if-Modified-Since: 浏览器再次请求服务器的时候，请求头会包含此字段，后面跟着在缓存中获得的最后修改时间。服务端收到此请求头发现有if-Modified-Since，则与被请求资源的最后修改时间进行对比，如果一致则返回304和响应报文头，浏览器只需要从缓存中获取信息即可。 从字面上看，就是说：从某个时间节点算起，是否文件被修改了：

1. 如果真的被修改：那么开始传输响应一个整体，服务器返回：200 OK
2. 如果没有被修改：那么只需传输响应 header，服务器返回：304 Not Modified

if-Unmodified-Since: 从字面上看, 就是说: 从某个时间点算起, 是否文件没有被修改

1. 如果没有被修改: 则开始`继续'传送文件: 服务器返回: 200 OK

2. 如果文件被修改: 则不传输,服务器返回: 412 Precondition failed (预处理错误)

   

这两个的区别是一个是修改了才下载一个是没修改才下载。 Last-Modified 说好却也不是特别好，因为如果在服务器上，一个资源被修改了，但其实际内容根本没发生改变，会因为 Last-Modified 时间匹配不上而返回了整个实体给客户端（即使客户端缓存里有个一模一样的资源）。为了解决这个问题，HTTP1.1推出了Etag。

#### Etag

Etag： 服务器响应请求时，通过此字段告诉浏览器当前资源在服务器生成的唯一标识（生成规则由服务器决定）

If-None-Match： 再次请求服务器时，浏览器的请求报文头部会包含此字段，后面的值为在缓存中获取的标识。服务器接收到次报文后发现 If-None-Match 则与被请求资源的唯一标识进行对比。

1. 不同，说明资源被改动过，则响应整个资源内容，返回状态码200。
2. 相同，说明资源无心修改，则响应header，浏览器直接从缓存中获取数据信息。返回状态码304.

但是实际应用中由于 Etag 的计算是使用算法来得出的，而算法会占用服务端计算的资源，所有服务端的资源都是宝贵的，所以就很少使用Etag了。

### 3、不同刷新执行过程

​	浏览器地址栏中写入URL，回车 浏览器发现缓存中有这个文件了，不用继续请求了，直接去缓存拿。

​	F5 是告诉浏览器，别偷懒，好歹去服务器看看这个文件是否有过期了。于是浏览器就发送一个请求带上 If-Modify-since。

​	Ctrl+F5 告诉浏览器，你先把你缓存中的这个文件给我删了，然后再去服务器请求个完整的资源文件下来。于是客户端就完成了强行更新的操作。

### 4、如何避免浏览器缓存

无法被浏览器缓存的请求： 

​	HTTP信息头中包含Cache-Control:no-cache，pragma:no-cache，或Cache-Control:max-age=0等告诉浏览器不用缓存的请求 

​	需要根据Cookie，认证信息等决定输入内容的动态请求是不能被缓存的 

​	经过HTTPS安全加密的请求（有人也经过测试发现，ie其实在头部加入Cache-Control：max-age信息，firefox在头部加入Cache-Control:Public之后，能够对HTTPS的资源进行缓存，参考《HTTPS的七个误解》） 

​	POST请求无法被缓存 

​	HTTP响应头中不包含Last-Modified/Etag，也不包含Cache-Control/Expires的请求无法被缓存 
