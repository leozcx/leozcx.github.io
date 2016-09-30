---
layout: post
title: content-type 浅析 
author: Leo
---

今天的一项工作是调用供应商提供的REST接口获取数据，供应商提供了一个文档，看起来还挺详细，记录了每个接口的url，参数列表，返回结果等；以为是个很简单的工作。于是向往常一样，打开poster，先模拟一下看看能不能调通。

像往常一样，把json设为content-type，在post body里设好json格式的参数，点击“发送”，却返回错误：“xx为空”。怎么回事，我明明在post
body里把xx设置了呀，就在那呢，为什么找不到呢？我意识到有可能是content-type的问题，后端服务不接受application/json，那还敢声称自己提供的是REST接口？

于是google，之前对content-type有些了解，知道有不同的content-type，比如application/json,
applicaiton/x-www-form-urlencoded...但是在什么情况下用哪个，以及在不同content-type情况下数据是怎么被发送给服务端的，这个却没有一个系统的认识，正好借这个机会，把这一套完整的弄清楚，备以后复查。

# HTTP 
目前广泛应用的是HTTP/1.1，分为2种报文：请求，响应

## 请求报文

|请求方法|空格|URL|空格|协议版本|回车|换行|
|c|d|

我们常用的请求方法为GET, POST, PUT, DELETE.

## 响应报文

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
