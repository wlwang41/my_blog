---
title: "http header for dummies"
date: 2014-07-16 23:31
---

由于最近工作有个功能是要到处excel给用户下载, 所以要用到`Content-Disposition`, 所以想顺便把http header里面常用的几个总结一下.

HTTP的headers是HTTP requests和responses的核心部分, 它携带了包括客户端浏览器的信息和请求的页面内容以及server部分信息的内容.能够通过HTTP的HEAD方法直接拿到.

对于HTTP Request而言, 它的结构第一行称为request line, 它包括HTTP方法, 请求路径以及协议三个部分.

对于HTTP Response而言, 它的结构第一行称为status line, 它包括协议和返回码两部分.

状态码大致可分为:
* 200's are used for successful requests.
* 300's are for redirections.
* 400's are used if there was a problem with the request.
* 500's are used if there was a problem with the server.

具体的内容可以点击[这里](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

其他的部分以键值对的形式存在每一行中.

## HTTP Request

### Host

HTTP请求必须发送给一个确定的IP地址, 但是由于大多数的服务器是可以host多个服务用同一个IP的, 所以HTTP请求必须清楚指明具体的域名.

    Host: wlwang41.github.io

### User-Agent

这个头可以包括以下信息:

* 浏览器的名称以及版本
* 操作系统的名称以及版本
* 默认的语言环境

这个可以用来判断用户使用的环境是手机还是PC, 然后呈现不同的页面.

    User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.1.5) Gecko/20091102 Firefox/3.5.5 (.NET CLR 3.5.30729)

### Accept-Language

这个头信息表明了用户的语言环境.

可以是多个语言环境, 用逗号隔开. 第一个语言为主要的偏好语言, `q`则表明了用户使用该语言的偏好(0, 1之间)

### Accept-Encoding

这个头信息表示浏览器能接受的压缩格式.

大多数的现代浏览器都支持gzip压缩, 所以一般这个字段都会有gzip. 这样webserver就能返回一个压缩后的数据, 节省带宽和请求时间.

    Accept-Encoding: gzip,deflate

### If-Modified-Since

浏览器可以通过这个字段来判断已经被cache的文档是否被修改过, 如果没有被修改过, 那么服务器将会返回"`304 Not Modified`", 并且没有内容, 浏览器就会读入缓存的数据. 但是这个还是会有一次HTTP请求.

    If-Modified-Since: Sat, 28 Nov 2009 06:38:19 GMT

### Cookie

以键值对的形式被服务器种在浏览器上.

### Referer

表明这个文档的来源, 比如在wlwang41.github.io里面点击了一个a标签, 跳到了另一个页面, 那么新打开的页面中的header中的Referer字段就应该为wlwang41.github.io.

**referer**实际上是拼错了, 正确的应该是**referrer**.

### Authorization

当一个页面需要认证的时候, 浏览器将会打开一个窗口让用户输入用户名密码, 然后浏览器将会再发一个HTTP请求, 这一次将会带上这个头部信息.

    Authorization: Basic bXl1c2VyOm15cGFzcw==

这个数据是被base64编码过的.

## HTTP Response

### Cache-Control

浏览器的cache策略. 如:

    Cache-Control: max-age=3600, public

"`public`"指的是任何人都可以缓存这个数据, "`max-age`"表明具体缓存的过期时间.

如果不希望有缓存则可以使用"`no-cache`"字段. 如:

    Cache-Control: no-cache

### Content-Type

这个字段表明返回文档的"`mime-type`", 浏览器将会根据这个类型来决定如何展示这个文档.比如:

    Content-Type: text/html; charset=UTF-8

其他的`mime-type`可以点击[这里](http://webdesign.about.com/od/multimedia/a/mime-types-by-content-type.htm).

### Content-Disposition

这个字段就表明浏览器不应该解析数据还是应该弹出下载框. 比如(合适的数据类型也应该告诉浏览器):

    Content-Type: application/zip
    Content-Disposition: attachment; filename="download.zip"

### Content-Length

文档的大小. 比如:

    Content-Length: 89123

下载的时候很好用, 可以用来指定进度条的进度.

### Etag

Etag也是与缓存策略相关的. 比如:

    Etag: "pub1259380237;gz"

web server可以每次返回都会带上这个头信息. 它的值可以使基于最近修改时间, 文件大小或者文件的校验码的值. 浏览器将会保存这个值, 当下一次请求这个文件时, 它将被作为头的一部分发送过去:

    If-None-Match: "pub1259380237;gz"

如果Etag的值匹配, 那么服务器将返回304并且没有具体的内容, 这样浏览器就能用缓存的值.

### Last-Modified

这个字段表明服务器最近的修改时间, 如:

    Last-Modified: Sat, 28 Nov 2009 03:50:37 GMT

浏览器通过发送`If-Modified-Since: Sat, 28 Nov 2009 06:38:19 GMT`来判断缓存是否过期.

### Location

需要跳转的地址. 如:

    HTTP/1.x 301 Moved Permanently
    ...
    Location: http://wlwang41.github.io/
    ...

### Content-Encoding

表明返回的数据的压缩格式, 如:

    Content-Encoding: gzip

----

参考文章: [HTTP Headers for Dummies](http://code.tutsplus.com/tutorials/http-headers-for-dummies--net-8039)
