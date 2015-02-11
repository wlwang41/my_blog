---
title: "python web背景知识"
date: 2015-02-11 16:58
---

我第一个接触的python的framework是tornado，它封装得好，写起来很顺手。
随着项目越做越多，学到的东西越多，见识到的东西越多，我发现自己还是有些东西没有弄清楚。

> 什么是web framework，什么是web server，什么是cgi，什么是wsgi，它们之间又有什么样的关系呢？

随着项目越做越多，学到的东西越多，见识到的东西越多，我发现自己弄清楚了。

## 背景

我正式做web开发的时候，已经是web2.0时代了，现在和以前的web有着很大的不同。
以前的web开发一般就是网站开发，那时候的网站都是静态页面组成的，web服务器只需要根据uri，把资源返回给浏览器就好了。
现在就不一样了，很多内容都是动态产生的，比如用户UGC的资源，有些也都是web服务，比如搜索等，web server不是简单地去
服务器拿一个HTML或者一张图片返回，而是需要执行一段代码，动态的产生结果。
大多数的HTTP server都是用c或者c++写的，它们并不能直接执行python代码。
因此，需要一个桥梁或者说接口让web server能够执行python代码，但是不是任何一个web server都支持现有的各种接口。
cgi/wsgi/fastcgi都是web server和应用程序之间通信的协议，桥梁。

## CGI

CGI是最老的也是几乎所有web server都支持的一种协议。
使用CGI和web server交流的程序，每一个请求都会新起一个python解释器程序去跑python代码最后销毁进程。
可想而知，这种情况是非常好资源并且慢的，几乎不会在生产环境中使用。
实际上，由于wsgi的优势，python世界中几乎都是走的wsgi协议和webserver交流。

## FastCGI

为了解决CGI每次都要新起一个解释进程，执行，销毁进程的性能为题，FastCGI的解决方法是本身后面跑着一个进程解释程序， 不需要每次都创建进程。
现在php世界的解决方案很多都是走的FastCGI，web server用的是nginx，后面跑一个php-fpm。

## WSGI

WSGI在FastCGI上面相当于做了一个把webserver那的http请求转成python的request对象，
然后应用程序输出的response对象转成http输出，对于这些协议来说，webserver是c端，应用程序是s端，WSGI还可以有middleware层，
在c端和s端之间，封装session等操作，由于WSGI直接把http数据转成了requst/response对象，
所以写python的web框架就不需要关心http协议的东西，也就很方便，同时只要支持WSGI的webserver和应用程序，
可以随便更换。有了WSGI，用python开发web程序才爽起来，也是为什么python有那么多web framework的原因。
一般来说，python的web framework只需要包括4个部分：

* Routing: Requests to function-call mapping with support for clean and dynamic URLs.
* Templates: Fast and pythonic built-in template engine and support for mako, jinja2 and cheetah templates.
* Utilities: Convenient access to form data, file uploads, cookies, headers and other HTTP-related metadata.
* Server: Built-in HTTP development server and support for paste, fapws3, bjoern, gae, cherrypy or any other WSGI capable HTTP server.



> 更多请参考:

> - [https://docs.python.org/2/howto/webservers.html](https://docs.python.org/2/howto/webservers.html)
> - [https://wiki.python.org/moin/WebProgramming](https://wiki.python.org/moin/WebProgramming)

