# Cookie 和 Session 关系和区别

​	HTTP 是无状态协议，说明它不能以状态来区分和管理请求和响应。也就是说，服务器单从网络连接上无从知道客户身份。

​	可是怎么办呢？就给客户端们颁发一个通行证吧，每人一个，无论谁访问都必须携带自己通行证。这样服务器就能从通行证上确认客户身份了。这就是 Cookie 的工作原理。

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621eec70c0418a2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 一、什么是 cookie

​	Cookie 是客户端保存用户信息的一种机制，用来记录用户的一些信息，实际上 Cookie 是服务器在**本地机器**上存储的一小段文本，并随着每次请求发送到服务器。

​	Cookie 技术通过请求和响应报文中写入 Cookie 信息来控制客户端的状态。

​	Cookie会根据响应报文里的一个叫做 Set-Cookie 的首部字段信息，通知客户端保存 Cookie。当下客户端再向服务端发起请求时，客户端会自动在请求报文中加入 Cookie 值之后发送出去.

​	之后服务端发现客户端发送过来的 Cookie 后，会检查是那个客户端发送过来的请求，然后对服务器上的记录，最后得到了之前的状态信息。

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621f0b5f29d7f7c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

客户端保存了 Cookie 之后的发起请求

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621f13fade59484?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621f407b4b82236?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上图很清晰地展示了发生 Cookie 交互的情景，HTTP 请求报文和响应报文的内容如图所示。

​	第一可以很明显的可出首部字段内没有 Cookie 的相关信息，其次也能看到 set-Cookie 里的信息，这就是服务器端生撑的 Cookei 信息。

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621f4051a540bbb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

看之后请求，请求报文里都自动发送 Cookie 信息了。

#### set-Cookie 的字段的属性

```properties
Set-Cookie: logcookie=3qjj; expires=Wed, 13-Mar-2019 12:08:53 GMT; Max-Age=31536000; path=/; domain=fafa.com;secure; HttpOnly;
```

以上面的 set-cookie 的例子，说一下set-cookie的属性

1.`logcookie=3qjj` 赋予 Cookie 的名称和值，logcookie 是名字 ，3qjj 是值

2.expires 是设置 cookie 有效期。当省略 expires 属性时，Cookie 仅在关闭浏览器之前有效。可以通过覆盖已过期的 Cookie，设置这个 Cookie 的过期时间是过去的时间，实现对客户端 Cookie 的实质性删除操作。

3.path 是限制指定 Cookie 的发送范围的文件目录。不过另有办法可避开这项限制，看来对其作为安全机制的效果不能抱有期待。

4.domain 通过 domain 属性指定的域名可以做到与结尾匹配一致。比如，指定 domain 是 fafa.com，除了fafa.com 那么 www.fafa.com 等都可以发送Cookie。

5.secure 设置web页面只有在 HTTPS 安全连接时，才可以发送 Cookie。HHTP 则不可以进行回收。

6.HttpOnly 它使 JavaScript 脚本无法获得 Cookie，通过上述设置，通常从 Web 页面内还可以对 Cookie 进行读取操作。但使用 JavaScript 的 document.cookie 就无法读取附加 HttpOnly 属性后的 Cookie 的内容了

### 二、Session 管理和 Cookie应用

#### 什么是 Session

​	我们可以把**客户端浏览器与服务器之间一系列交互的动作**称为一个 Session。 从这个语义出发，我们会提到Session持续的时间，会提到在Session过程中进行了什么操作等等。

​	在Java中是通过调用HttpServletRequest的getSession方法(使用true作为参数)创建的。 创建Session的同时，**服务器会为该Session生成唯一的session id**， 这个session id在随后的请求中会被用来重新获得已经创建的Session，Session被创建之后，就可以调用Session相关的方法往Session中增加内容了， 而这些内容只会保存在服务器中，发到客户端的只有session id。当客户端再次发送请求的时候，会将这个session id带上， 服务器接受到请求之后就会依据session id找到相应的Session，从而再次使用Session。

​	当客户端的 Cookie 被禁用或出现问题时，可以把 Session ID 附着在 URL 中(**URL重写的技术**)或者以 POST 方法把请求发送给服务器，这样再通过 Session ID 就能跨页使用 Session 变量了。

![img](https://user-gold-cdn.xitu.io/2018/3/13/1621f6d2880ac3ab?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 三、Cookie 与 Session 的区别

1. Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息，也是实现Session的一种方式。；
2. Session是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中。