---
title: "没有外网权限的机器装python模块"
date: 2014-11-26 15:18
---

由于公司服务器大多数是没有外网权限的，也没有自己的pip源，所以要在服务器上装python的第三方模块相对来说比较复杂.

实际上，pip有一个功能就是直接把一个项目的所有依赖下载下来，并且不安装：

    pip install --download /YOUR/DOWNLOAD/PATH celery

上面这个命令就能把celery以及它的依赖包全部现在到本地指定路径中。

把这些包都传到服务器上，然后在服务器上运行：

    pip install --no-index -f /YOUR/PACKAGES/PATH celery

这样就完成了一次python包的安装。

当然，这种方式非常麻烦，如果要装多台服务器的多个python包将做很多无用功。

因此我利用fabric写了一个简单的脚本，只需要在config.py里面写入本地下载包的路径，和服务器的host
然后运行 `fab install_pys:celery,tornado,requests` 就能完成安装.

也可以手动一步一步安装，具体命令通过 `fab -l` 就能看到。

项目地址请戳[这里](https://github.com/wlwang41/install_pys)。

当然搭建一个pypi镜像才是最方便的。

近期会开始做一些统一部署方案的工作，以后的python项目可能就会用virtualenv或者docker做隔离，并且搭建一个内部的pypi镜像，
然后规范requirements.txt，那么装环境就只需要 `pip install -r requirements.txt` 就能完成一个项目的环境环境配置了。
