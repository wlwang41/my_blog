---
title: "command tricks"
date: 2014-07-10 16:24
---

* 查看目录结构

        tree

    > mac默认是没有tree命令的，所以可以有2种办法:

    1. `brew install tree`
    2. `find . -print | sed -e 's;[^/]*/;|____;g;s;____|; |;g'`

* 查看代码有多少行

        find . -name "*.py" | xargs wc -l

* 删除所有的pyc文件

        find . -name "*.pyc" -delete

* 返回到上一个目录

        cd -

* 生成随机数

        jot -r [number_of_numbers] [lower_limit] [upper_limit]

* 显示当前谁登陆到了你的系统

        w

* 将一个字符串打印很多次

        yes [string]

* 更好看的git log

        git log --graph --abbrev-commit --decorate --date=relative --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)' --all

* 更好看的git log(带详细信息)

        git log --graph --abbrev-commit --decorate --date=relative --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)' --all -p

* curl相关命令

    * 直接输出到命令行

            curl http://www.centos.org

    * 输出到指定文件

            curl -o mygettext.html http://www.gnu.org/software/gettext/manual/gettext.html

        or 直接用url里面的文件名

            curl -O http://www.gnu.org/software/gettext/manual/gettext.html

    * 允许redirects

            curl -L http://www.google.com

    * 断点续传

            curl -C - -O http://www.gnu.org/software/gettext/manual/gettext.html

        > 也可指定续传的位置

    * 限定下载速度

            curl --limit-rate 1000B -O http://www.gnu.org/software/gettext/manual/gettext.html

    * 需要认证

            curl -u username:password URL

    * 使用代理

            curl -x proxysever.test.com:3128 http://google.co.in

    * 发邮件

            curl --mail-from blah@test.com --mail-rcpt foo@test.com smtp://mailserver.com

        输入后，就可以开始写邮件内容， .表示结束

            Subject: Testing
            This is a test mail
            .

    * HTTP Method

            curl --request GET 'http://www.somedomain.com/'
            curl --request POST 'http://www.somedomain.com/'
            curl --request DELETE 'http://www.somedomain.com/'
            curl --request PUT 'http://www.somedomain.com/'
    * 传入参数

            curl --request POST 'http://www.somedomain.com/login/' --data 'username=myusername&password=mypassword'
            curl --data-urlencode "date=April 1" 'http://example.com/form.cgi'

        上传文件:

            curl --request POST 'http://127.0.0.1:8008/api/trace_file' -F 'trace_file=@journal.txt;type=text/plain'

    * 带上头部信息

            curl --request GET 'http://www.somedomain.com/user/info/' --header 'sessionid:1234567890987654321'

        需要请求返回带上头信息:

            curl --request GET 'http://somedomain.com/' --include

        `-I` 则是只显示头信息

    * 获取详细返回结果

            curl -v 'http://somedomain.com/'

        还不够详细？

            curl --trace output.txt 'http://somedomain.com/'

        or

            curl --trace-ascii output.txt 'http://somedomain.com/'

    * 提供referer信息

            curl --referer 'http://www.example.com http://www.example.com'

    * UA

            curl --user-agent "[User Agent]" [URL]

    * cookie

            curl --cookie "name=xxx" www.example.com

    * 带host访问

            curl -H 'Host: project1.loc' 'http://127.0.0.1/something'

* 命令行打开任一程序(限OSX)

        open /Applications/Safari.app/

    or 打开一个目录

        open .

* 粘贴复制(限OSX)

    将一个文件内容拷贝到clipboard:

        pbcopy < blogpost.txt

    将clipboard的内容追加到文件:

        pbpaste >> tasklist.txt

* 截图(限OSX)

    屏幕截图保存到image.png并发送到Mail中

        screencapture -C -M image.png

    鼠标选择区域截图并保存到clipboard

        screencapture -c -W

    10秒之后截图并用Preview打开

        screencapture -T 10 -P image.png

    鼠标截取任意区域内容,保存为pdf

        screencapture -s -t pdf image.pdf

* 说出任意内容(限OSX)

        say 'You love me.'

* OSX读写ntfs格式移动硬盘

        mkdir -p /Volumes/1 && sudo mount -t ntfs /dev/disk1s1 /Volumes/1

    之后操作1文件夹就行了

    弹出:

        sudo umount /dev/disk1s1
