---
title: "POST发送数据方式"
date: 2014-11-26 17:47
---

POST方法发送数据的方式有2种，一种是 `application/x-www-form-urlencoded`, 另外一种就是 `multipart/form-data` .


## application/x-www-form-urlencoded

对于这种方式，传输给服务端的HTTP的body的信息是一个巨大的字符串，就像：

    ☁  ~  curl -v -X POST -d "fizz=buzz&buzz=fizz" http://requestb.in/1e8sq8a1
    * Hostname was NOT found in DNS cache
    *   Trying 50.16.239.160...
    * Connected to requestb.in (127.0.0.1) port 80 (#0)
    > POST /1e8sq8a1 HTTP/1.1
    > User-Agent: curl/7.37.1
    > Host: requestb.in
    > Accept: */*
    > Content-Length: 19
    > Content-Type: application/x-www-form-urlencoded
    >
    * upload completely sent off: 19 out of 19 bytes
    < HTTP/1.1 200 OK
    < Connection: keep-alive
    * Server gunicorn/18.0 is not blacklisted
    < Server: gunicorn/18.0
    < Date: Wed, 26 Nov 2014 14:15:14 GMT
    < Content-Type: text/html; charset=utf-8
    < Content-Length: 2
    < Sponsored-By: https://www.runscope.com
    < Via: 1.1 vegur
    <
    * Connection #0 to host requestb.in left intact
    ok%

这里request的 `Content-Type` 就是 `application/x-www-form-urlencoded` .
这种方式对于要传输的巨大的字符串中是有键值对构成，键和值之间由 `=` 隔开，每组键值对之间由 `&` 隔开.
但是这种这方式对于数据有一定的要求：

* 空格将会被替换成 `+` 
* 所有非字母数字的字符将会被替换成 `%HH` ，也就是一个百分号和2个16进制的值来代表该字符的ASCII码
* 折行将会被替换成 `CR LF` ( `%0D%0A` )

由于有这样的规则，每一个非字母数字的字节将会被替换成3个字节，对于大的二级制文件，3倍的数据负载显得非常没有效率.

所以对于传输文件，非ASCII的数据，以及二级制的数据，就得用 `multipart/form-data`


## multipart/form-data

下面的例子可以看到，请求的 `Content-Type` 就是 `multipart/form-data` ，里面的每组数据由 `boundary` 隔开.

    ☁  ~  curl -v -X POST -F "fizz=buzz&buzz=fizz" http://requestb.in/1e8sq8a1
    * Hostname was NOT found in DNS cache
    *   Trying 50.16.239.160...
    * Connected to requestb.in (127.0.0.1) port 80 (#0)
    > POST /1e8sq8a1 HTTP/1.1
    > User-Agent: curl/7.37.1
    > Host: requestb.in
    > Accept: */*
    > Content-Length: 153
    > Expect: 100-continue
    > Content-Type: multipart/form-data; boundary=------------------------f5926011c527fb7e
    >
    < HTTP/1.1 100 Continue
    < HTTP/1.1 200 OK
    < Connection: keep-alive
    * Server gunicorn/18.0 is not blacklisted
    < Server: gunicorn/18.0
    < Date: Wed, 26 Nov 2014 14:26:07 GMT
    < Content-Type: text/html; charset=utf-8
    < Content-Length: 2
    < Sponsored-By: https://www.runscope.com
    < Via: 1.1 vegur
    <
    * Connection #0 to host requestb.in left intact
    ok%

这种方式传输的数据，每一部分都是MIME的信息，没一部分由 `boundary` 字符串隔开，每一部分都可以有自己的MIME头信息，比如 `Content-Type` ，`Content-Disposition` 。
特别是 `Content-Disposition` ，这个头信息可以指定键值对中的键名，而键值对中的值就是MIME的信息主体。
通过w3c上的例子可以很清楚的理解这种协议：

    <form action="http://server.com/cgi/handle" enctype="multipart/form-data" method="post">
        <P>
            What is your name? <INPUT type="text" name="submit-name"><BR>
            What files are you sending? <INPUT type="file" name="files"><BR>
        </p>
        <INPUT type="submit" value="Send"> <INPUT type="reset">
    </form>

这里是一段表单提交的html代码，当用户在text的input标签中输入了Larry，在file的input标签中选择了file1.txt的时候，user agent将会回传这样的数据:

    Content-Type: multipart/form-data; boundary=AaB03x

    --AaB03x
    Content-Disposition: form-data; name="submit-name"

    Larry
    --AaB03x
    Content-Disposition: form-data; name="files"; filename="file1.txt"
    Content-Type: text/plain

    ... contents of file1.txt ...
    --AaB03x--

在 `Content-Type` 中指明数据传输的类型是 `multipart/form-data` ，并且boundary字符串为AaB03x。
每一部分MIME信息都有上面指定的边界字符串隔开，并且每一部分MIME信息都可以有自己的头，如 `Content-Disposition: form-data; name="submit-name"` ，表明submit-name字段的值为Larry。
如果传输的是文件，文件名可以通过 `Content-Disposition` 中的 `filename` 字段来指明。

如果在file的input标签里面又传了一张图片，那么user agent将会返回这样的数据：

    Content-Type: multipart/form-data; boundary=AaB03x

    --AaB03x
    Content-Disposition: form-data; name="submit-name"

    Larry
    --AaB03x
    Content-Disposition: form-data; name="files"
    Content-Type: multipart/mixed; boundary=BbC04y

    --BbC04y
    Content-Disposition: file; filename="file1.txt"
    Content-Type: text/plain

    ... contents of file1.txt ...
    --BbC04y
    Content-Disposition: file; filename="file2.gif"
    Content-Type: image/gif
    Content-Transfer-Encoding: binary

    ...contents of file2.gif...
    --BbC04y--
    --AaB03x--

多文件的情况，外层的 `Content-Type` 将会是 `multipart/mixed` 并且指定了不同文件之间的boundary字符串。

`multipart/form-data` 可以解决非字母数字组成的大字符串的传输带来的3倍的负载，但是也不是说所有数据传输都应该用它。
因为对于一般的数据而言，大量的boundary字符也是一种性能浪费。
所以，如果你需要传输包含非字母数字字符的数据(二进制数据)，就用 `multipart/form-data` ，否则就用 `application/x-www-form-urlencoded` 。

> 更多请参考[RFC2388](http://www.ietf.org/rfc/rfc2388.txt)，以及[w3c上的解释](http://www.w3.org/TR/html401/interact/forms.html#h-17.13.4)
