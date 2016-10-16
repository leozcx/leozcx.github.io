---
layout: post
title: content-type 浅析 
author: Leo
---

今天的一项工作是调用供应商提供的REST接口获取数据，供应商提供了一个文档，看起来还挺详细，记录了每个接口的url，参数列表，返回结果等；以为是个很简单的工作。于是向往常一样，打开poster，先模拟一下看看能不能调通。

REST API通常都是json格式的输出与输出，于是把json设为content-type，在post
body里设好json格式的参数，点击“发送”，却返回错误：“xx为空”。从字面上看，是API需要某个参数，但参数没有提供，这是怎么回事呢，那个参数明明在post
body里设置了呀，为什么找不到呢？隐约感觉到有可能是content-type的问题，后端服务并不认json。

于是google，之前对content-type有些了解，知道有不同的content-type，比如application/json, text/html, 
applicaiton/x-www-form-urlencoded...但是在什么情况下用哪个，以及在不同content-type情况下数据是怎么被发送给服务端的，这个却没有一个系统的认识，正好借这个机会，把这一套完整的弄清楚，备以后复查。

# HTTP 
首先是HTTP报文格式，目前广泛应用的是HTTP/1.1，分为2种报文：请求，响应。

## 请求报文

![请求报文格式](https://d17oy1vhnax1f7.cloudfront.net/items/0G0V0I2C3E1L1W332Z3S/2012072810301161.3n1R0T0z1u2l.png)

可以将请求报文分为3个部分：请求行，请求头部，请求数据（包体）；请求行只有一行，请求头部可有有多行，请求数据也可以有多行；请求头部与数据之间有一个空行来分隔。

- 请求行

包含3个部分：请求方法，url, HTTP版本，比如访问某个网址，发的请求方法一般是GET, url就是网址，HTTP版本目前是1.1
我们常用的请求方法为GET, POST, PUT, DELETE.

- 请求头部

头部是冒号分隔的键值对，跟在请求行之后；可以有多个头部，每个头部就是一行，以回车(CR),换行(LF)结尾；最后跟一个空行表示头部字段结束。常见的请求头部有：

  -  Accept: 用于和Server协商以什么格式的内容交互，比如text/html,application/xhtml+xml
  -  Cache-Control: 用于表示浏览器的缓存策略；通常在request header中的Cache-Control作用不大
  - Cookie: 浏览器保存的对应于访问网址的cookie，是多个以分号分隔的键-值对，键-值对以等号分隔；每次请求浏览器都会把保存的所有cookie都包含在这，这对很多时候是对网络带宽的一种浪费
  - Content-Length: 请求数据的长度，以byte为单位 
  - Content-Type: 请求数据的格式，比如Content-Type: application/x-www-form-urlencoded
  - Host: 指明要访问的网址域名，也可以包含端口号；该字段是必填字段
  - Origin: 请求是从哪个地址发起的，浏览器在进行跨域请求时会自动设置该字段
  - Referer: 用户从哪个网址访问到当前网址，常见的例子是通常搜索引擎访问某个网站时，Referer就会被设为搜索引擎的地址
  - Uer-Agent: 指明用户浏览器信息，比如Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.143 Safari/537.36

- 请求数据

GET类型的请求没有请求数据，POST/PUT有。


## 响应报文

![响应报文格式](https://cl.ly/2M2C2r123u1V/20131107163544468.png)

与请求报文类似，也可以分为3个部分：响应行，响应头部，响应数据。

- 响应行

包含3个部分：协议版本，状态码，状态码描述。常见的状态码有：

2XX Sucess:

 - 200：OK
 - 201: Created
 - 202: Accepted

3XX Redirection

 - 301: Moved Permanently
 - 304: Not Modified
 - 307: Temporary Redirect

4XX Client Error

  - 400: Bad Request
  - 401: Unauthorized; 实际上表示未认证（未登陆）
  - 403: Forbidden; 实际是未授权
  - 404: Not Found

5XX Server Error

  - 500: Internal Server Error; 通用错误
  - 502: Bad Gateway; 反向代理与后端服务器发生交互问题时，比如超时，会返回此错误码

- 响应头部

  - Access-Control-Allow-Origin: 是否允许跨域请求，比如Access-Control-Allow-Origin: * ; 以前用jsonp来处理跨域访问，现在推荐用这种方式来支持跨域请求
  - Cache-Control: 指明浏览器应该如何处理cache,可以包含若干个值，每个值以逗号分隔
    - public
    - private
    - no-cache
    - no-store
    - max-age=seconds
  - Content-Encoding: 响应数据以何种压缩方式压缩，比如gzip
  - Content-Length: 响应数据的长度，以byte为单位
  - Content-Type: 响应数据的类型，比如Content-Type: text/html; charset=utf-8
  - ETag: 可以认为是响应数据的指纹，如果ETag相同，则认为内容没有发生变化，比如ETag: "737060cd8c284d8af7ad3082f209582d"
  - Expires: 指定日期之后内容被认为失效，比如Expires: Thu, 01 Dec 1994 16:00:00 GMT
  - Set-Cookie: 设置cookie，比如Cookie: UserID=JohnDoe; Max-Age=3600; Version=1
  
  可以看到，有些头部是请求，响应报文所共有的，比如Content-Type，Cache-Control，而大部分是没有共用的。

- 响应数据

在发送HTTP报文时，数据是放在请求包体中的，但协议并没有规定我们必须要以什么样的格式来发送数据，比如我可以用xml，也可以用json，也可以用无结构的文本，这就要求我们告诉服务器：我发送的数据的格式是什么，服务器好作相应的解析，这就是Content-Type的作用。

# Content-Type

HTTP规范中规定Content-Type的格式为 type/subtype[;parameter] , type预定了以下几种：
- application
- audio
- image
- message
- multipart
- text
- video
- x-token
这几种足以应对平时遇到的大多数情况了，如果实在不够，也可以扩展，只要服务器端支持就可以，扩展的type必须以 'X-'开头，表示这不是标准的type。
我们在ajax中常用的 `application/json;charset=utf-8`就是这种格式。

我们在提供数据时通常用POST请求，使用的Content-Type通常有3种：

## application/x-www-form-urlencoded

这是比较常用的Content-Type(在ajax应用中application/json更推荐使用)，浏览器表单提交时默认的就是这种，JQuery
ajax默认也是这种。它实际上把数据以key-value的形式放在了最终的url中，而没有放在请求包体。看下面这个例子：

```
POST http://www.example.com HTTP/1.1
Content-Type: application/x-www-form-urlencoded;charset=utf-8

title=test&sub%5B%5D=1&sub%5B%5D=2&sub%5B%5D=3
```
可以看到，不同的key-value之间以"&"连接，并对特殊字符作了转义。

## multipart/form-data

application/x-www-form-urlencoded可以满足基本的表单提交需求，但如果要上传文件怎么办呢？这就要用到multipart/form-data了，它和application/x-www-form-urlencoded都是form支持的Content-Type。与application/x-www-form-urlencoded不同的是，multipart/form-data不会把数据接到url中，而是会放到请求包体，不同的数据以一个boundary分隔，这个boundary必须不包含在直接要发送的数据内容中，所以一般是一个很长的字符串，下面是一个例子：

```
POST http://www.example.com HTTP/1.1
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA

------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"

title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png

PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```

## application/json

在ajax世界中，application/json是最推荐使用的一种，angular的$http默认使用的就是这种，各大服务器端框架也都能很好地支持它。它的好处是可读性好，可以支持复杂的数据结构。它也是把数据放在请求包体中。

好，回到今天遇到的问题：原因是坑爹的文档并没有注明Content-Type要设为application/x-www-form-urlencoded，而我用的工具默认是application/json，难怪后端找不到必须的参数。

# 参考资料

- [https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4](https://www.w3.org/TR/html401/interact/forms.html#h-17.13.4)
- [https://imququ.com/post/four-ways-to-post-data-in-http.html](https://imququ.com/post/four-ways-to-post-data-in-http.html)
- [https://en.wikipedia.org/wiki/List_of_HTTP_status_codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
