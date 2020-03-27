## HTTP 请求、响应报文格式

### HTTP 请求报文格式

##### 由请求行、请求头部、请求正文3部分组成：

![img](https://pic002.cnblogs.com/images/2012/426620/2012072810301161.png)

#### 请求行：

请求方法、URL、协议版本（空格分隔）

#### 请求头部：

Host 						 接受请求的服务器地址，可以是IP:端口号，也可以是域名

User-Agent 			 发送请求的应用程序名称

Connection 			指定与连接相关的属性，如 Connection:Keep-Alive

Accept-Charset 	  通知服务端可以发送的编码格式

Accept-Encoding    通知服务端可以发送的数据压缩格式

Accept-Language   通知服务端可以发送的语言

### HTTP 响应报文格式

响应行、响应头、空行、响应体。

```html
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Type: text/html;charset=utf-8

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>Document</title>
</head>
<body>
    <p>this is http response</p>
</body>
</html>
```

Server 							 服务器应用程序软件的名称和版本

Content-Type 				响应正文的类型（是图片还是二进制字符串）

Content-Length 			响应正文长度

Content-Charset 		   响应正文使用的编码

Content-Encoding 		响应正文使用的数据压缩格式

Content-Language 	   响应正文使用的语言
