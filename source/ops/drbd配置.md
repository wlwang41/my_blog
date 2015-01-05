---
title: "DRBD配置"
date: 2015-01-04 17:02
---

这段时间正在考虑如何激励大家去做code review的事情，我想到的是在gitlab上做二次开发，加入打分的功能。

首先master分支作为线上分支，任何想要上线的feature都要提merge request。每一个merge request要想被接受，必须得到至少5分。
一般开发者最多只能打2分，master身份的人可以打5分，所有人如果觉得代码写得不太好，可以打负分。希望通过这个打分机制，让大家都能去review别人的代码。

开发完了之后问了问公司的gitlab的HA方案，得到的结果比较让人吃惊，于是我们决定研究一下如何做gitlab的高可用。

简单的来说，gitlab这个项目分为ruby on rails和git2个部分，rails那边主要就是web端展示和一些网页上的操作，git端就是git的各种操作。
rails在这部分和git这部分的通讯走的是用sidekiq(redis)作的消息队列，数据库用的postgres。gitlab官方推荐了一些HA的[方案](https://about.gitlab.com/high-availability/)，我们选择的是它推荐的双机HA方案。

这个方案就是用[DRBD](http://www.drbd.org/)这个中间件做了磁盘的网络备份。它允许用户在远程机器上建立一个本地块设备的实时镜像。它在linux内核2.6.33版本以及以上都是内置的一个模块。
它的同步策略有3种，我们采用的Protocol C，也就是备机同步完成才算写操作完成，这也是官方推荐的配置。

## 服务器配置

* 系统: Red Hat Enterprise Linux Server release 6.5 (Santiago) ( `cat /etc/issue` )
* 内核版本: 2.6.32-431.5.1.el6.x86_64 ( `uname -r` )
* DRBD版本: drbd-8.4.0.tar.gz
* 机器ip: 10.31.84.109和10.31.84.206

## 依赖安装

运行下面命令，安装编译所需依赖：

    yum install gcc libxslt kernel-headers kernel-devel-`uname -r` make wget rpm-build flex

## 下载安装源码包

在指定路径下如 `/opt/resource` 下运行：

    wget http://oss.linbit.com/drbd/8.4/drbd-8.4.0.tar.gz && tar xzvf drbd-8.4.0.tar.gz
    cd drbd-8.4.0
    ./configure --prefix=/opt/drbd --localstatedir=/var --sysconfdir=/etc --with-km
    make && make install
    cp /opt/drbd/etc/rc.d/init.d/drbd /etc/rc.d/init.d
    chkconfig --add drbd
    chkconfig drbd on

## 编译模块

    cd drbd
    make clean all
    make KDIR=/lib/modules/`uname -r`
    cp drbd.ko /lib/modules/`uname -r`/kernel/lib/
    depmod

## 准备分区

运行 `fdisk -l` 得到机器的硬盘信息，比如有一块没有分区的盘/dev/xvdb。现在就对这块盘分区作为同步的磁盘。

运行 `fdisk /dev/xvdb` ，输入m会得到帮助信息，这里我们需要新建分区，所以需要选择n。
接着选择是主分区还是拓展分区，这里选主分区，也就是p。
然后选择分区号，由于这是块裸盘，所以选1.
紧接着选择开始和结束的柱面，我们只分一个区，所以都用默认值就好了。
最后输入w，完成分区。

运行 `partx /dev/xvdb` ，让内核重新读取分区，这个时候再运行 `fdisk -l` 或者 `cat /proc/partitions` 就能看到刚分的区了。

这个步骤需要在2个机器上都去执行。

## DRBD配置

打开/opt/drbd/etc/drbd.d/global_common.conf文件，确保在global中usage-count yes配置，以及common中的net里面有protocal C。

然后在/opt/drbd/etc/drbd.d/中创建一个r0.res的文件，输入如下信息:

    resource r0 {
      on host_84_109 {
        device    /dev/drbd1;
        disk      /dev/xvdb1;
        address   10.31.84.109:7789;
        meta-disk internal;
      }
      on host_84_206 {
        device    /dev/drbd1;
        disk      /dev/xvdb1;
        address   10.31.84.206:7789;
        meta-disk internal;
      }
    }

要注意主机名和服务器的主机名一致。

## 建立资源

首先载入drbd模块

    modprobe drbd

确认drbd模块载入

    lsmod | grep drbd

清空分区数据，如果下面配文件系统失败，很有可能是这里需要清一下

    dd if=/dev/zero of=/dev/sda3 bs=1M count=100

建立资源r0

    drbdadm create-md r0

## 启动drbd服务

在2台机器上运行下面命令启动服务：

    service drbd start

可以通过 `/opt/drbd/sbin/drbd-overview` 命令查看集群状态。

在一台上运行 `drbdadm -- --overwrite-data-of-peer primary r0` 来进行初始化的同步。

## 创建文件系统比挂载

运行：`mkfs.ext4 /dev/drbd1` 创建文件系统，然后选择要挂载到的地址： `mount /dev/drbd1 /opt/gitlab`

## 切换角色

可以通过命令 `drbdadm secondary r0` 和 `drbdadm primary r0` 来做角色的切换，要确保没有2个角色同时为主。

## 测试

在当前的主节点上挂载点上创建一些文件，如 `touch test.txt`，然后要确保切换角色之前要讲主节点的资源不在被使用。
运行 `umount /opt/gitlab` .

通过命令将主节点降为此节点 `drbdadm secondary r0` .

在次节点上运行命令升为主节点 `drbdadm primary r0` ，
在次节点上挂载资源 `mount /dev/drbd1 /opt/gitlab` ，此时cd进去就能看到在另一台机器创建的文件了。
