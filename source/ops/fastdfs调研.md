---
title: "fastdfs调研"
date: 2014-08-12 14:33
---

架构组第一件事情就是搭建一个文件系统来代替搜狐的云存储。
我们考虑过重写[Beansdb](https://github.com/douban/beansdb)，但是由于工期问题和使用量最后没有考虑使用这个方案，
[金庸](http://jinyong.me/)同学调研了一圈，最终决定使用[FastDFS](https://code.google.com/p/fastdfs/)。

其实我对它还是有点不放心，虽然有很多公司都在用这个FS，
但是介绍它的文档都是分散的，没有人去维护一个官方的文档，给我的感觉很不正式，万一哪天作者不去维护了怎么办呢？
由于我没能提出另一个可靠的方案，所以也就决定使用它了。希望不要成为和以前选用cassandra作为主数据库一样的黑历史。
这篇文章主要整理一些碎片的文档，希望能够慢慢完善最终成为我们组内使用FastDFS的一个wiki。

## FastDFS安装

安装请参考[这里](http://www.zrwm.com/?cat=131)。

## FastDFS概述

通过[Google](http://search.aol.com/aol/search?q=fastdfs&s_it=opensearch)来搜FastDFS，
会看到前四个结果分别是[google code](http://code.google.com/p/fastdfs/)和[sourceforge](http://sourceforge.net/projects/fastdfs/)上托管的源码。
很奇怪为什么作者不在Github上托管一份，这2个更新和管理代码太不方便，更不谈googlecode被墙了。

第五个结果是它的[论坛](http://bbs.chinaunix.net/forum-240-1.html)，很多资料和使用经验应该都会出自这里。哎，写个官方使用文档是他妈有多难？

后面2篇文章是[uc博客](http://tech.uc.cn/?p=221)上面别人分享的，写得很好，讲清楚了FastDFS的原理，概述这一小节将会是对这2篇文章的整理。

### 什么是FastDFS

FastDFS是一个开源的轻量级分布式文件系统，它解决了大数据量存储和负载均衡等问题。
特别适合以中小文件（建议范围： **4KB** 到 **500MB** ）为载体的在线服务，如相册网站、视频网站等等。

### FastDFS架构

FastDFS服务端有三个角色：跟踪服务器（tracker server）、存储服务器（storage server）和客户端（client）。

* tracker server: 跟踪服务器，主要做调度工作，起负载均衡的作用。
在内存中记录集群中所有存储组和存储服务器的状态信息，是客户端和数据服务器交互的枢纽。
相比GFS中的master更为精简，不记录文件索引信息，占用的内存量很少。

* storage server: 存储服务器（又称：存储节点或数据服务器），文件和文件属性（meta data）都保存到存储服务器上。
Storage server直接利用OS的文件系统调用管理文件。

* client server: 客户端，作为业务请求的发起方，通过专有接口，使用TCP/IP协议与跟踪器服务器或存储节点进行数据交互。

<img src="http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS%E6%9E%B6%E6%9E%84%E5%9B%BE1.png" style="width: 500px;" />

FastDFS这样的架构十分轻量，没有Master角色，使用对等结构，组间数据是户备，很方便的加入新的组来实现平行扩展，各个节点之前的通讯协议都是TCP/IP的。

### 上传机制

<img src="http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS%E4%B8%8A%E4%BC%A0%E6%B5%81%E7%A8%8B.png" style="width: 500px;" />

### 组件同步时间管理

当一个文件上传成功后，客户端马上发起对该文件下载请求（或删除请求）时，tracker是如何选定一个适用的存储服务器呢？

其实每个存储服务器都需要定时将自身的信息上报给tracker，这些信息就包括了本地同步时间（即同步到的最新文件的时间戳）。
而tracker根据各个存储服务器的上报情况，就能够知道刚刚上传的文件，在该存储组中是否已完成了同步。同步信息上报如下图：

<img src="http://tech.uc.cn/wp-content/uploads/2012/08/%E7%AE%A1%E7%90%86%E5%90%8C%E6%AD%A5%E6%97%B6%E9%97%B4.png" style="width: 500px;" />


### 下载机制

<img src="http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS%E4%B8%8B%E8%BD%BD%E6%B5%81%E7%A8%8B.png" style="width: 500px;" />

### FID

文件索引结构为：`g1/M00/00/00/CgoYr1PoeaaAQhnkAAzrjmXcV8I399.jpg`
它是客户端上传文件后存储服务器返回给客户端，用于以后访问该文件的索引信息。
文件索引信息包括：组名，虚拟磁盘路径，数据两级目录，文件名。

其中：

* 组名: 文件上传后所在的存储组名称，在文件上传成功后有存储服务器返回，需要客户端自行保存。
* 虚拟磁盘路径: 存储服务器配置的虚拟路径，与磁盘选项store_path* 对应。
* 数据两级目录: 存储服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。
* 文件名: 与文件上传时不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。

具体的使用场景为：

1. 通过组名tracker能够很快的定位到客户端需要访问的存储服务器组，并将选择合适的存储服务器提供客户端访问；

2. 存储服务器根据“文件存储虚拟磁盘路径”和“数据文件两级目录”可以很快定位到文件所在目录，并根据文件名找到客户端需要访问的文件。

<img src="http://tech.uc.cn/wp-content/uploads/2012/08/%E6%96%87%E4%BB%B6%E6%99%BA%E8%83%BD%E5%AE%9A%E4%BD%8D1.png" style="width: 500px;" />

### 总结

* FastDFS只有三个角色；且跟踪服务器和存储服务器均不存在单点。

* 跟踪服务器被动的接收存储服务器汇报，对存储服务器进行分组管理；并为客户端选定适用的存储服务器。同一存储服务器可以同时向多台跟踪服务器汇报状态信息。

* 存储服务器组内所有存储服务器是对等关系，存储的数据一一对应且相同；所有的存储服务器均是同时在线服务，极大的提高的服务器的使用率，分担了数据访问压力。

## FastDFS使用技巧

### 下载时返回可读文件名

文件被上传到FastDFS后Storage服务端将返回的文件索引（FID），
其中文件名是根据FastDFS自定义规则重新生成的，而不是原始文件名，例如： group2/M00/00/89/eQ6h3FKJf_PRl8p4AUz4wO8tqaA688.apk

使用http下载时如不加处理，显示给用户的文件名会是这样的eQ6h3FKJf_PRl8p4AUz4wO8tqaA688.apk，这样的用户体验很不好。
由于FastDFS不会存储原始文件名，也没有提供恢复原始文件名的方法，所以需要应用系统自己想办法恢复原始文件名。

解决方法:

1. 应用系统在上传文件到FastDFS成功时将原始文件名和“文件索引（FID）”保存下来（例如：保存到数据库）。

2. 用户点击下载的时用Nginx的域名和FID拼出url，然后在url后面增加一个参数，指定原始文件名。
例如： http://121.14.161.48:9030/group2/M00/00/89/eQ6h3FKJf_PRl8p4AUz4wO8tqaA688.apk?attname=filename.apk

3. 在Nginx上进行如下配置，这样Nginx就会截获url中的参数attname，在Http响应头里面加上字段 `Content-Disposition "attachment;filename=$arg_attname"`

        location /group2/M00 {
            root /data/store/data;
            if ($arg_attname ~ "^(.*).apk") {
                    add_header Content-Disposition "attachment;filename=$arg_attname";
            }
            ngx_fastdfs_module;
        }

4. 浏览器发现响应头里面有 `Content-Disposition "attachment;filename=$arg_attname"` 时，就会把文件名显示成filename指定的名称。

    > 在使用fdfs_nginx_module 1.15时可以直接在http 请求中加入 `filename=real_file_name` 即可以返回文件的原始名称 `real_file_name` ，不需要添加nginx添加相应头的配置。

### 自带命令

* fdfs_delete_file: 删除文件，`/opt/apps/fdfs/bin/fdfs_delete_file <config_file> <file_id>`

* fdfs_download_file: 下载文件，`/opt/apps/fdfs/bin/fdfs_download_file <config_file> <file_id> <local_filename>`

* fdfs_file_info: 获取某个文件信息，`/opt/apps/fdfs/bin/fdfs_file_info <config_file> <file_id>`

* fdfs_monitor: 监控，查看storage server的状态`/opt/apps/fdfs/bin/fdfs_monitor <config_file>`
也可以用这个命令来摘除无效的storage节点，`/opt/apps/fdfs/bin/fdfs_monitor <config_file> delete <group_name> <storage_ip>`
注意：如果被删除的storage server的状态是ACTIVE，也就是该storage server还在线上服务的情况下，是无法删除掉的。

* fdfs_storaged: 用来启动或是重启或是关闭server，`/opt/apps/fdfs/bin/fdfs_storaged <config_file> [start | stop | restart]`

* fdfs_trackerd: 用来启动或是重启或是关闭server，`/opt/apps/fdfs/bin/fdfs_trackerd <config_file> [start | stop | restart]`

* fdfs_upload_file: 上传文件，`/opt/apps/fdfs/bin/fdfs_upload_file <config_file> <local_filename>`

### 配置详解

#### tracker.conf

    disabled=false  # 这个配置文件是否不生效

    bind_addr=  # 是否绑定IP，后面为绑定的IP地址

    port=22122  # 提供服务的端口

    connect_timeout=30  # 连接超时时间，针对socket套接字函数connect

    network_timeout=60  # tracker server的网络超时，单位为秒。发送或接收数据时，如果在超时时间后还不能发送或接收数据，则本次网络通信失败。

    base_path=/home/yuqing/fastdfs  # base_path 目录地址(根目录必须存在,子目录会自动创建)
    # 附目录说明:
        tracker server目录及文件结构：
        ${base_path}
          |__data
          |     |__storage_groups.dat：存储分组信息
          |     |__storage_servers.dat：存储服务器列表
          |__logs
                |__trackerd.log：tracker server日志文件

            数据文件storage_groups.dat和storage_servers.dat中的记录之间以换行符（\n）分隔，字段之间以西文逗号（,）分隔。
            storage_groups.dat中的字段依次为：
                1. group_name：组名
                2. storage_port：storage server端口号

        storage_servers.dat中记录storage server相关信息，字段依次为：
            1. group_name：所属组名
            2. ip_addr：ip地址
            3. status：状态
            4. sync_src_ip_addr：向该storage server同步已有数据文件的源服务器
            5. sync_until_timestamp：同步已有数据文件的截至时间（UNIX时间戳）
            6. stat.total_upload_count：上传文件次数
            7. stat.success_upload_count：成功上传文件次数
            8. stat.total_set_meta_count：更改meta data次数
            9. stat.success_set_meta_count：成功更改meta data次数
            10. stat.total_delete_count：删除文件次数
            11. stat.success_delete_count：成功删除文件次数
            12. stat.total_download_count：下载文件次数
            13. stat.success_download_count：成功下载文件次数
            14. stat.total_get_meta_count：获取meta data次数
            15. stat.success_get_meta_count：成功获取meta data次数
            16. stat.last_source_update：最近一次源头更新时间（更新操作来自客户端）
            17. stat.last_sync_update：最近一次同步更新时间（更新操作来自其他storage server的同步）

    max_connections=256  # 系统提供服务时的最大连接数

    accept_threads=1  # 允许的线程数

    work_threads=4  # V2.0引入的这个参数，工作线程数，通常设置为CPU数，要<= max_connections

    # 上传组(卷) 的方式 0:轮询方式 1: 指定组 2: 平衡负载(选择最大剩余空间的组(卷)上传)
    # 这里如果在应用层指定了上传到一个固定组,那么这个参数被绕过
    store_lookup=2

    store_group=group2  # 当上一个参数设定为1 时 (store_lookup=1，即指定组名时)，必须设置本参数为系统中存在的一个组名。如果选择其他的上传方式，这个参数就没有效了。

    # 选择哪个storage server 进行上传操作(一个文件被上传后，这个storage server就相当于这个文件的storage server源，会对同组的storage server推送这个文件达到同步效果)
    # 0: 轮询方式
    # 1: 根据ip 地址进行排序选择第一个服务器（IP地址最小者）
    # 2: 根据优先级进行排序（上传优先级由storage server来设置，参数名为upload_priority）
    store_server=0

    # 选择storage server 中的哪个目录进行上传。storage server可以有多个存放文件的base path（可以理解为多个磁盘）。
    # 0: 轮流方式，多个目录依次存放文件
    # 2: 选择剩余空间最大的目录存放文件（注意：剩余磁盘空间是动态的，因此存储到的目录或磁盘可能也是变化的）
    store_path=0

    # 选择哪个 storage server 作为下载服务器
    # 0: 轮询方式，可以下载当前文件的任一storage server
    # 1: 哪个为源storage server 就用哪一个 (前面说过了这个storage server源 是怎样产生的) 就是之前上传到哪个storage server服务器就是哪个了
    download_server=0

    # storage server 上保留的空间，保证系统或其他应用需求空间。可以用绝对值或者百分比（V4开始支持百分比方式）。
    #(指出 如果同组的服务器的硬盘大小一样,以最小的为准,也就是只要同组中有一台服务器达到这个标准了,这个标准就生效,原因就是因为他们进行备份)
    reserved_storage_space = 10%

    log_level=info  # 选择日志级别

    run_by_group=  # 操作系统运行FastDFS的用户组 (不填 就是当前用户组,哪个启动进程就是哪个)

    run_by_user=  # 操作系统运行FastDFS的用户 (不填 就是当前用户,哪个启动进程就是哪个)

    allow_hosts=*  # 可以连接到此 tracker server 的ip范围（对所有类型的连接都有影响，包括客户端，storage server）

    # 同步或刷新日志信息到硬盘的时间间隔，单位为秒
    # 注意：tracker server 的日志不是时时写硬盘的，而是先写内存。
    sync_log_buff_interval = 10

    # 检测 storage server 存活的时间隔，单位为秒。
    # storage server定期向tracker server 发心跳，
    # 如果tracker server在一个check_active_interval内还没有收到storage server的一次心跳，那边将认为该storage server已经下线。
    # 所以本参数值必须大于storage server配置的心跳时间间隔。通常配置为storage server心跳时间间隔的2倍或3倍。
    check_active_interval = 120

    # 线程栈的大小。FastDFS server端采用了线程方式。更正一下，tracker server线程栈不应小于64KB，不是512KB。
    # 线程栈越大，一个线程占用的系统资源就越多。如果要启动更多的线程（V1.x对应的参数为max_connections， V2.0为work_threads），可以适当降低本参数值。
    thread_stack_size=1MB

    storage_ip_changed_auto_adjust=true  # 这个参数控制当storage server IP地址改变时，集群是否自动调整。注：只有在storage server进程重启时才完成自动调整。

    # V2.0引入的参数。存储服务器之间同步文件的最大延迟时间，缺省为1天。根据实际情况进行调整
    # 注：本参数并不影响文件同步过程。本参数仅在下载文件时，判断文件是否已经被同步完成的一个阀值（经验值）
    storage_sync_file_max_delay = 86400

    # V2.0引入的参数。存储服务器同步一个文件需要消耗的最大时间，缺省为300s，即5分钟。
    # 注：本参数并不影响文件同步过程。本参数仅在下载文件时，作为判断当前文件是否被同步完成的一个阀值（经验值）
    storage_sync_file_max_time = 300

    use_trunk_file = false  # V3.0引入的参数。是否使用小文件合并存储特性，缺省是关闭的。

    # V3.0引入的参数。
    # trunk file分配的最小字节数。比如文件只有16个字节，系统也会分配slot_min_size个字节。
    slot_min_size = 256

    # V3.0引入的参数。
    # 只有文件大小<=这个参数值的文件，才会合并存储。如果一个文件的大小大于这个参数值，将直接保存到一个文件中（即不采用合并存储方式）。
    slot_max_size = 16MB

    # V3.0引入的参数。
    # 合并存储的trunk file大小，至少4MB，缺省值是64MB。不建议设置得过大。
    trunk_file_size = 64MB

    trunk_create_file_advance = false  # 是否提前创建trunk file。只有当这个参数为true，下面3个以trunk_create_file_打头的参数才有效。

    trunk_create_file_time_base = 02:00  # 提前创建trunk file的起始时间点（基准时间），02:00表示第一次创建的时间点是凌晨2点。

    trunk_create_file_interval = 86400  # 创建trunk file的时间间隔，单位为秒。如果每天只提前创建一次，则设置为86400

    # 提前创建trunk file时，需要达到的空闲trunk大小
    # 比如本参数为20G，而当前空闲trunk为4GB，那么只需要创建16GB的trunk file即可。
    trunk_create_file_space_threshold = 20G

    trunk_init_check_occupying = false  #trunk初始化时，是否检查可用空间是否被占用

    # 是否无条件从trunk binlog中加载trunk可用空间信息
    # FastDFS缺省是从快照文件storage_trunk.dat中加载trunk可用空间，
    # 该文件的第一行记录的是trunk binlog的offset，然后从binlog的offset开始加载
    trunk_init_reload_from_binlog = false

    use_storage_id = false  # 是否使用server ID作为storage server标识

    # use_storage_id 设置为true，才需要设置本参数
    # 在文件中设置组名、server ID和对应的IP地址，参见源码目录下的配置示例：conf/storage_ids.conf
    storage_ids_filename = storage_ids.conf

    # 存储从文件是否采用symbol link（符号链接）方式
    # 如果设置为true，一个从文件将占用两个文件：原始文件及指向它的符号链接。
    store_slave_file_use_link = false

    rotate_error_log = false  # 是否定期轮转error log，目前仅支持一天轮转一次

    error_log_rotate_time=00:00   # error log定期轮转的时间点，只有当rotate_error_log设置为true时有效

    # error log按大小轮转
    # 设置为0表示不按文件大小轮转，否则当error log达到该大小，就会轮转到新文件中
    rotate_error_log_size = 0

    # 以下是关于http的设置了 默认编译是不生效的 要求更改 #WITH_HTTPD=1 将 注释#去掉 再编译
    # 关于http的应用 说实话 不是很了解 没有见到 相关说明 ,望 版主可以完善一下 以下是字面解释了
    http.disabled=false  # HTTP服务是否不生效
    http.server_port=8080  # HTTP服务端口


    #use "#include" directive to include http other settiongs
    ##include http.conf  # 如果加载http.conf的配置文件 去掉第一个

### storage.conf

    disabled=false  # 同上

    group_name=group1  # 指定此storage server所在组(卷)

    bind_addr=  # 同上

    # bind_addr通常是针对server的。当指定bind_addr时，本参数才有效。
    # 本storage server作为client连接其他服务器（如tracker server、其他storage server），是否绑定bind_addr。
    client_bind=true

    port=23000  # storage server服务端口

    connect_timeout=30  # 同上

    network_timeout=60  # 同上

    heart_beat_interval=30  # 心跳间隔时间，单位为秒 (这里是指主动向tracker server 发送心跳)

    stat_report_interval=60  # storage server向tracker server报告磁盘剩余空间的时间间隔，单位为秒。

    base_path=/home/yuqing/fastdfs  # 同上

    max_connections=256  # 同上

    work_threads=4  # 同上

    # V2.0引入本参数。设置队列结点的buffer大小。工作队列消耗的内存大小 = buff_size * max_connections
    # 设置得大一些，系统整体性能会有所提升。
    # 消耗的内存请不要超过系统物理内存大小。另外，对于32位系统，请注意使用到的内存不要超过3GB
    buff_size = 256KB

    # V2.09引入本参数。设置为true，表示不使用操作系统的文件内容缓冲特性。
    # 如果文件数量很多，且访问很分散，可以考虑将本参数设置为true
    disk_rw_direct = false

    disk_rw_separated = true  # V2.0引入本参数。磁盘IO读写是否分离，缺省是分离的。

    # V2.0引入本参数。针对单个存储路径的读线程数，缺省值为1。
    # 读写分离时，系统中的读线程数 = disk_reader_threads * store_path_count
    # 读写混合时，系统中的读写线程数 = (disk_reader_threads + disk_writer_threads) * store_path_count
    disk_reader_threads = 1

    # V2.0引入本参数。针对单个存储路径的写线程数，缺省值为1。
    # 读写分离时，系统中的写线程数 = disk_writer_threads * store_path_count
    # 读写混合时，系统中的读写线程数 = (disk_reader_threads + disk_writer_threads) * store_path_count
    disk_writer_threads = 1

    # 同步文件时，如果从binlog中没有读到要同步的文件，休眠N毫秒后重新读取。0表示不休眠，立即再次尝试读取。
    # 出于CPU消耗考虑，不建议设置为0。如何希望同步尽可能快一些，可以将本参数设置得小一些，比如设置为10ms
    sync_wait_msec=200

    sync_interval=0  # 同步上一个文件后，再同步下一个文件的时间间隔，单位为毫秒，0表示不休眠，直接同步下一个文件。

    sync_start_time=00:00

    sync_end_time=23:59  # 上面二个一起解释。允许系统同步的时间段 (默认是全天) 。一般用于避免高峰同步产生一些问题而设定

    # 同步完N个文件后，把storage的mark文件同步到磁盘
    # 注：如果mark文件内容没有变化，则不会同步
    write_mark_file_freq=500

    store_path_count=1  # 存放文件时storage server支持多个路径（例如磁盘）。这里配置存放文件的基路径数目，通常只配一个目录。

    #store_path1=/home/yuqing/fastdfs2
    # 逐一配置store_path个路径，索引号基于0。注意配置方法后面有0,1,2 ......，需要配置0到store_path - 1。
    # 如果不配置base_path0，那边它就和base_path对应的路径一样。
    store_path0=/home/yuqing/fastdfs

    # FastDFS存储文件时，采用了两级目录。这里配置存放文件的目录个数 (系统的存储机制,大家看看文件存储的目录就知道了)
    # 如果本参数只为N（如：256），那么storage server在初次运行时，会自动创建 N * N 个存放文件的子目录。
    subdir_count_per_path=256

    # tracker_server 的列表 要写端口的哦 (再次提醒是主动连接tracker_server )
    # 有多个tracker server时，每个tracker server写一行
    tracker_server=10.62.164.84:22122
    tracker_server=10.62.245.170:22122

    log_level=info  # 同上

    run_by_group=  # 同上

    run_by_user=  # 同上

    # 允许连接本storage server的IP地址列表 （不包括自带HTTP服务的所有连接）
    # 可以配置多行，每行都会起作用
    allow_hosts=*

    #  文件在data目录下分散存储策略。
    # 0: 轮流存放，在一个目录下存储设置的文件数后（参数file_distribute_rotate_count中设置文件数），使用下一个目录进行存储。
    # 1: 随机存储，根据文件名对应的hash code来分散存储。
    file_distribute_path_mode=0

    # 当上面的参数file_distribute_path_mode配置为0（轮流存放方式）时，本参数有效。
    # 当一个目录下的文件存放的文件数达到本参数值时，后续上传的文件存储到下一个目录中。
    file_distribute_rotate_count=100

    fsync_after_written_bytes=0  # 当写入大文件时，每写入N个字节，调用一次系统函数fsync将内容强行同步到硬盘。0表示从不调用fsync

    # 同步或刷新日志信息到硬盘的时间间隔，单位为秒
    # 注意：storage server 的日志信息不是时时写硬盘的，而是先写内存。
    sync_log_buff_interval=10

    # 同步binglog（更新操作日志）到硬盘的时间间隔，单位为秒
    # 本参数会影响新上传文件同步延迟时间
    sync_binlog_buff_interval=60

    # 把storage的stat文件同步到磁盘的时间间隔，单位为秒。
    # 注：如果stat文件内容没有变化，不会进行同步
    sync_stat_file_interval=300

    # 线程栈的大小。FastDFS server端采用了线程方式。
    # 对于V1.x，storage server线程栈不应小于512KB；对于V2.0，线程栈大于等于128KB即可。
    # 线程栈越大，一个线程占用的系统资源就越多。
    # 对于V1.x，如果要启动更多的线程（max_connections），可以适当降低本参数值。
    thread_stack_size=512KB

    upload_priority=10  # 本storage server作为源服务器，上传文件的优先级，可以为负数。值越小，优先级越高。这里就和 tracker.conf 中store_server= 2时的配置相对应了

    # 是否检测上传文件已经存在。如果已经存在，则不存在文件内容，建立一个符号链接以节省磁盘空间。
    # 这个应用要配合FastDHT 使用，所以打开前要先安装FastDHT
    # 1或yes 是检测，0或no 是不检测
    check_file_duplicate=0

    # 文件去重时，文件内容的签名方式：
    ## hash： 4个hash code
    ## md5：MD5
    file_signature_method=hash

    key_namespace=FastDFS  # 当上个参数设定为1 或 yes时 (true/on也是可以的) ， 在FastDHT中的命名空间。

    keep_alive=0  # 与FastDHT servers 的连接方式 (是否为持久连接) ，默认是0（短连接方式）。可以考虑使用长连接，这要看FastDHT server的连接数是否够用。

    use_access_log = false  # 是否将文件操作记录到access log

    rotate_access_log = false  # 是否定期轮转access log，目前仅支持一天轮转一次

    access_log_rotate_time=00:00  # access log定期轮转的时间点，只有当rotate_access_log设置为true时有效

    rotate_error_log = false  # 是否定期轮转error log，目前仅支持一天轮转一次

    error_log_rotate_time=00:00  # error log定期轮转的时间点，只有当rotate_error_log设置为true时有效

    # access log按文件大小轮转
    # 设置为0表示不按文件大小轮转，否则当access log达到该大小，就会轮转到新文件中
    rotate_access_log_size = 0

    # error log按文件大小轮转
    # 设置为0表示不按文件大小轮转，否则当error log达到该大小，就会轮转到新文件中
    rotate_error_log_size = 0

    file_sync_skip_invalid_record=false  # 文件同步的时候，是否忽略无效的binlog记录

    http.disabled=false

    http.server_port=8888

    http.trunk_size=256KB  # http.trunk_size表示读取文件内容的buffer大小（一次读取的文件内容大小），也就是回复给HTTP client的块大小。

    # storage server上web server域名，通常仅针对单独部署的web server。这样URL中就可以通过域名方式来访问storage server上的文件了，
    # 这个参数为空就是IP地址的方式。
    http.domain_name=

    #use "#include" directive to include HTTP other settiongs
    ##include http.conf

### 缓存

缓存的方案选取的是在tracker的机器上安装 `ngx_cache_purge` 模块来提供缓存。

修改nginx配置如下:

    worker_processes  4;                  #根据CPU核心数而定
    events {
        worker_connections  65535;        #最大链接数
        use epoll;                        #新版本的Linux可使用epoll加快处理性能
    }
    http {
        #设置缓存参数
        server_names_hash_bucket_size 128;
        client_header_buffer_size 32k;
        large_client_header_buffers 4 32k;
        client_max_body_size 300m;
        sendfile        on;
        tcp_nopush      on;
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 90;
        proxy_send_timeout 90;
        proxy_read_timeout 90;
        proxy_buffer_size 16k;
        proxy_buffers 4 64k;
        proxy_busy_buffers_size 128k;
        proxy_temp_file_write_size 128k;
        #设置缓存存储路径、存储方式、分配内存大小、磁盘最大空间、缓存期限
        proxy_cache_path /var/cache/nginx/proxy_cache levels=1:2 keys_zone=http-cache:500m max_size=10g inactive=30d;
        proxy_temp_path /var/cache/nginx/proxy_cache/tmp;

        #设置group1的服务器
        upstream fdfs_group1 {
            server 172.16.1.203:8080 weight=1 max_fails=2 fail_timeout=30s;
            server 172.16.1.204:8080 weight=1 max_fails=2 fail_timeout=30s;
        }
        server {
            #设置服务器端口
            listen       8080;
            #设置group1的负载均衡参数
            location /group1/M00 {
                proxy_next_upstream http_502 http_504 error timeout invalid_header;
                proxy_cache http-cache;
                proxy_cache_valid  200 304 12h;
                proxy_cache_key $uri$is_args$args;
                proxy_pass http://fdfs_group1;
                expires 30d;
            }
        }
        location ~ /purge(/.*) {
            allow 127.0.0.1;
            allow 172.16.1.0/24;
            deny all;
            proxy_cache_purge http-cache  $1$is_args$args;
        }
    }

重启nginx，上传一张图片，然后去nginx的cache目录查看是否有缓存文件，要清除缓存只需在url中group之前加上`purge`。

> 但是，都要读文件，只是换一个路径这样的缓存意义何在?

## Fastdfs-zyc所实现的service

- BaseService  基类，仅包含Session

- FileDataService
     - 根据组名获取文件列表
     - 根据ip和fileId去数据库查找DownloadFileRecord
     - 保存DownloadFileRecord

- JobService   重点，所有其他service的数据都是由这个服务从服务端读取数据保存在数据库上
     - 按分钟更新组数据
     - 按小时更新组数据
     - 按天更新组数据
     - 从日志中读取数据保存到数据库

- MonitorService
     - 列出组信息
     - 列出所有组
     - 根据组名列出storage
     - …

- StructureService
     - listStorageTopLine
     - listStorageAboutFile

- UserService 监控系统的用户控制相关

- WarningService
     - 添加删除被通知用户
     - 添加删除预警warning
     - 更新warning数据

## FastDFS 同步机制

*参考自作者博客[FastDFS How to -- 同步机制](http://blog.chinaunix.net/uid-20315669-id-1967081.html)*

在FastDFS的服务器端配置文件中，bind_addr这个参数用于需要绑定本机IP地址的场合。只有这个参数和主机特征相关，其余参数都是可以统一配置的。在不需要绑定本机的情况下，为了便于管理和维护，建议所有tracker server的配置文件相同，同组内的所有storage server的配置文件相同。

tracker server的配置文件中没有出现storage server，而storage server的配置文件中会列举出所有的tracker server。这就决定了storage server和tracker server之间的连接由storage server主动发起，storage server为每个tracker server启动一个线程进行连接和通讯，这部分的通信协议请参阅《FastDFS HOWTO -- Protocol》中的“2. storage server to tracker server command”部分。

tracker server会在内存中保存storage分组及各个组下的storage server，并将连接过自己的storage server及其分组保存到文件中，以便下次重启服务时能直接从本地磁盘中获得storage相关信息。storage server会在内存中记录本组的所有服务器，并将服务器信息记录到文件中。tracker server和storage server之间相互同步storage server列表：

1. 如果一个组内增加了新的storage server或者storage server的状态发生了改变，tracker server都会将storage server列表同步给该组内的所有storage server。以新增storage server为例，因为新加入的storage server主动连接tracker server，tracker server发现有新的storage server加入，就会将该组内所有的storage server返回给新加入的storage server，并重新将该组的storage server列表返回给该组内的其他storage server；
2. 如果新增加一台tracker server，storage server连接该tracker server，发现该tracker server返回的本组storage server列表比本机记录的要少，就会将该tracker server上没有的storage server同步给该tracker server。

同一组内的storage server之间是对等的，文件上传、删除等操作可以在任意一台storage server上进行。文件同步只在同组内的storage server之间进行，采用push方式，即源服务器同步给目标服务器。以文件上传为例，假设一个组内有3台storage server A、B和C，文件F上传到服务器B，由B将文件F同步到其余的两台服务器A和C。我们不妨把文件F上传到服务器B的操作为源头操作，在服务器B上的F文件为源头数据；文件F被同步到服务器A和C的操作为备份操作，在A和C上的F文件为备份数据。同步规则总结如下：

1. 只在本组内的storage server之间进行同步；
2. 源头数据才需要同步，备份数据不需要再次同步，否则就构成环路了；
3. 上述第二条规则有个例外，就是新增加一台storage server时，由已有的一台storage server将已有的所有数据（包括源头数据和备份数据）同步给该新增服务器。

**关于storage server的7个状态**

storage server有7个状态

- FDFS_STORAGE_STATUS_INIT      :初始化，尚未得到同步已有数据的源服务器
- FDFS_STORAGE_STATUS_WAIT_SYNC :等待同步，已得到同步已有数据的源服务器
- FDFS_STORAGE_STATUS_SYNCING   :同步中
- FDFS_STORAGE_STATUS_DELETED   :已删除，该服务器从本组中摘除（注：本状态的功能尚未实现）
- FDFS_STORAGE_STATUS_OFFLINE   :离线
- FDFS_STORAGE_STATUS_ONLINE    :在线，尚不能提供服务
- FDFS_STORAGE_STATUS_ACTIVE    :在线，可以提供服务

当storage server的状态为FDFS_STORAGE_STATUS_ONLINE时，当该storage server向tracker server发起一次heart beat时，tracker server将其状态更改为FDFS_STORAGE_STATUS_ACTIVE。

组内新增加一台storage server A时，由系统自动完成已有数据同步，处理逻辑如下：

1. storage server A连接tracker server，tracker server将storage server A的状态设置为FDFS_STORAGE_STATUS_INIT。storage server A询问追加同步的源服务器和追加同步截至时间点，如果该组内只有storage server A或该组内已成功上传的文件数为0，则没有数据需要同步，storage server A就可以提供在线服务，此时tracker将其状态设置为FDFS_STORAGE_STATUS_ONLINE，否则tracker server将其状态设置为FDFS_STORAGE_STATUS_WAIT_SYNC，进入第二步的处理；
2. 假设tracker server分配向storage server A同步已有数据的源storage server为B。同组的storage server和tracker server通讯得知新增了storage server A，将启动同步线程，并向tracker server询问向storage server A追加同步的源服务器和截至时间点。storage server B将把截至时间点之前的所有数据同步给storage server A；而其余的storage server从截至时间点之后进行正常同步，只把源头数据同步给storage server A。到了截至时间点之后，storage server B对storage server A的同步将由追加同步切换为正常同步，只同步源头数据；
3. storage server B向storage server A同步完所有数据，暂时没有数据要同步时，storage server B请求tracker server将storage server A的状态设置为FDFS_STORAGE_STATUS_ONLINE；
4. 当storage server A向tracker server发起heart beat时，tracker server将其状态更改为FDFS_STORAGE_STATUS_ACTIVE。

## FastDFS使用的通信协议
**参考自作者博客[FastDFS HOWTO -- Protocol](http://blog.chinaunix.net/uid-20315669-id-1967079.html)**

FastDFS采用TCP/IP协议，package包含header和body, header和body都可能为空

header格式：

    @ TRACKER_PROTO_PKG_LEN_SIZE bytes package length
    @ 1 byte command
    @ 1 byte status
    note:
    TRACKER_PROTO_PKG_LEN_SIZE (8) bytes number buff is Big-Endian bytes

body格式:

按照发送方和接收方角色不同可以分为5大类：

### 1. Common Command 通用命令
**FDFS_PROTO_CMD_QUIT**

    # function: notify server connection will be closed （通知服务器连接将会中断）
    # request body: none (no body part)
    # response: none (no header and no body)

**FDFS_PROTO_CMD_ACTIVE_TEST**

    # function: active test （测试是否active）
    # request body: none
    # response body: none

### 2. storage server 发送给tracker server的命令
这些commond的response 是 TRACKER_PROTO_CMD_STORAGE_RESP

**TRACKER_PROTO_CMD_STORAGE_JOIN**

    # function: storage join to tracker (将storage加入到tracker的track列表)
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN + 1 bytes: group name
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage http server port
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: path count
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: subdir count per path
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: upload priority
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: join time (join timestamp)
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: up time (start timestamp)
        @ FDFS_VERSION_SIZE bytes: storage server version
        @ FDFS_DOMAIN_NAME_MAX_SIZE bytes: domain name of the web server on the storage server
        @ 1 byte: init flag ( 1 for init done)
        @ 1 byte: storage server status
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: tracker server count excluding current tracker
    # response body:
        @ FDFS_IPADDR_SIZE bytes: sync source storage server ip address
    # memo: return all storage servers in the group only when storage servers changed or return none（仅当storage server生效才返回所有的storage server,否则返回空）

**TRACKER_PROTO_CMD_STORAGE_BEAT**

    # function: heart beat （心跳）
    # request body: none or storage stat info （返回空或者storage状态信息）
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total upload count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success upload count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total set metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success set metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total delete count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success delete count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total download count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success download count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total get metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success get metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total create link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success create link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total delete link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success delete link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: last source update timestamp
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: last sync update timestamp
       @TRACKER_PROTO_PKG_LEN_SIZE bytes:  last synced timestamp
       @TRACKER_PROTO_PKG_LEN_SIZE bytes:  last heart beat timestamp
    # response body: same to command TRACKER_PROTO_CMD_STORAGE_JOIN
    # memo: storage server sync it's stat info to tracker server only when storage stat info changed（仅当storage server的状态信息改变时才将其同步到tracker）

**TRACKER_PROTO_CMD_STORAGE_REPORT**

    # function: report disk usage(报告硬盘使用情况)
    # request body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total space in MB（硬盘总体积）
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: free space in MB （空闲大小）
    # response body: same to command TRACKER_PROTO_CMD_STORAGE_JOIN

**TRACKER_PROTO_CMD_STORAGE_REPLICA_CHG**

    # function: replica new storage servers which maybe not exist in the tracker server （？？？）
    # request body: n * (1 + FDFS_IPADDR_SIZE) bytes, n >= 1. One storage entry format:
        @ 1 byte: storage server status
        @ FDFS_IPADDR_SIZE bytes: storage server ip address
    # response body: none

**TRACKER_PROTO_CMD_STORAGE_SYNC_SRC_REQ**

    # function: source storage require sync. when add a new storage server, the existed storage servers in the same group will ask the tracker server to tell the source storage server which will sync old data to it（当新添加一个storage server时，同组的其他storage server会向tracker请求告诉新的storage有数据要同步给他）
    # request body:
        @ FDFS_IPADDR_SIZE bytes: dest storage server (new storage server) ip address （新加入的storage server的ip地址）
    # response body: none or
        @ FDFS_IPADDR_SIZE bytes: source storage server ip address（源storage server的ip地址）
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: sync until timestamp
    # memo: if the dest storage server not do need sync from one of storage servers in the group, the response body is emtpy（若新storage server不需要源storage server的同步，返回空）

**TRACKER_PROTO_CMD_STORAGE_SYNC_DEST_REQ**

    # function: dest storage server (new storage server) require sync（新加入的storage 向tracker请求同步）
    # request body: none
    # response body: none or
        @ FDFS_IPADDR_SIZE bytes: source storage server ip address
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: sync until timestamp
    # memo: if the dest storage server not do need sync from one of storage servers in the group, the response body is emtpy（若新storage server不需要源storage server的同步，返回空）

**TRACKER_PROTO_CMD_STORAGE_SYNC_NOTIFY**

    # function: new storage server sync notify(新storage同步通告)
    # request body:
        @ FDFS_IPADDR_SIZE bytes: source storage server ip address
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: sync until timestamp
    # response body: same to command TRACKER_PROTO_CMD_STORAGE_JOIN

### 3.client发送给tracker server的命令

以下两个命令的response是TRACKER_PROTO_CMD_SERVER_RESP

**TRACKER_PROTO_CMD_SERVER_LIST_GROUP**

    # function: list all groups （列出所有的group）
    # request body: none
    # response body: n group entries, n >= 0, the format of each entry:
     @ FDFS_GROUP_NAME_MAX_LEN+1 bytes: group name
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: free disk storage in MB
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server count
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server http port
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: active server count
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: current write server index
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: store path count on storage server
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: subdir count per path on storage server


**TRACKER_PROTO_CMD_SERVER_LIST_STORAGE**

    # function: list storage servers of a group（列出某个组的所有storage server）
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: the group name to query(要列的组名)
    # response body: n storage entries, n >= 0, the format of each entry:
       @ 1 byte: status
       @ FDFS_IPADDR_SIZE bytes: ip address
       @ FDFS_DOMAIN_NAME_MAX_SIZE  bytes : domain name of the web server
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: source storage server ip address
       @ FDFS_VERSION_SIZE bytes: storage server version
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: join time (join in timestamp)
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: up time (start timestamp)
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total space in MB
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: free space in MB
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: upload priority
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: store path count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: subdir count per path
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: current write path[
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage http port
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total upload count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success upload count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total set metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success set metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total delete count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success delete count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total download count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success download count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total get metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success get metadata count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total create link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success create link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: total delete link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: success delete link count
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: last source update timestamp
       @ TRACKER_PROTO_PKG_LEN_SIZE bytes: last sync update timestamp
       @TRACKER_PROTO_PKG_LEN_SIZE bytes:  last synced timestamp
       @TRACKER_PROTO_PKG_LEN_SIZE bytes:  last heart beat timestamp

以下命令response是 TRACKER_PROTO_CMD_SERVICE_RESP

**TRACKER_PROTO_CMD_SERVICE_QUERY_STORE_WITHOUT_GROUP_ONE**

    # function: query which storage server to store file(查询向哪一个storage server保存文件)
    # request body: none
    # response body:
     @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
     @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address
     @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
     @1 byte: store path index on the storage server

**TRACKER_PROTO_CMD_SERVICE_QUERY_STORE_WITHOUT_GROUP_ALL**

    # function: query which storage server to store file
    # request body: none
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address (* multi)
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port (*multi)
        @1 byte: store path index on the storage server

**TRACKER_PROTO_CMD_SERVICE_QUERY_STORE_WITH_GROUP_ONE**

    # function: query which storage server to store file
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
        @1 byte: store path index on the storage server

**TRACKER_PROTO_CMD_SERVICE_QUERY_STORE_WITH_GROUP_ALL**

    # function: query which storage server to store file
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address  (* multi)
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port   (* multi)
        @1 byte: store path index on the storage server


**TRACKER_PROTO_CMD_SERVICE_QUERY_FETCH**

    # function: query which storage server to download the file（查询向哪个storage server 请求下载文件）
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port

**TRACKER_PROTO_CMD_SERVICE_QUERY_FETCH_ALL**

    # function: query all storage servers to download the file
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ FDFS_IPADDR_SIZE - 1 bytes: storage server ip address
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: storage server port
        @ n * (FDFS_IPADDR_SIZE - 1) bytes:  storage server ip addresses, n can be 0

### 4. Storage server向Storage Server发送的命令
该类命令的response 是STORAGE_PROTO_CMD_RESP

**STORAGE_PROTO_CMD_SYNC_CREATE_FILE**

    # function: sync new created file （同步新创建的文件）
    # request body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: filename bytes
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file size/bytes
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes : filename
        @ file size bytes: file content
    # response body: none

**STORAGE_PROTO_CMD_SYNC_DELETE_FILE**

    # function: sync deleted file（同步删除的文件）
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body: none

**STORAGE_PROTO_CMD_SYNC_UPDATE_FILE**

    # function: sync updated file （同步更新的文件）
    # request body: same to command STORAGE_PROTO_CMD_SYNC_CREATE_FILE
    # respose body: none


### client向storage server 发送的命令
该类命令的response是 STORAGE_PROTO_CMD_RESP

**STORAGE_PROTO_CMD_UPLOAD_FILE**

    # function: upload file to storage server(上传文件到storage server)
    # request body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: meta data bytes
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file size
        @ meta data bytes: each meta data seperated by \x01,
                            name and value seperated by \x02
        @ file size bytes: file content
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename

**STORAGE_PROTO_CMD_UPLOAD_SLAVE_FILE**

    # function: upload slave file to storage server
    # request body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: master filename length
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file size
        @ FDFS_FILE_PREFIX_MAX_LEN bytes: filename prefix
        @ FDFS_FILE_EXT_NAME_MAX_LEN bytes: file ext name, do not include dot (.)
        @ master filename bytes: master filename
        @ file size bytes: file content
    # response body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename

 **STORAGE_PROTO_CMD_DELETE_FILE**

    # function: delete file from storage server(从storage server中删除文件)
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body: none

**STORAGE_PROTO_CMD_SET_METADATA**

    # function: delete file from storage server（从storage server中删除文件）
    # request body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: filename length
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: meta data size
        @ 1 bytes: operation flag,
            'O' for overwrite all old metadata（O, 覆盖所有metadata）
            'M' for merge, insert when the meta item not exist, otherwise update it（M, 插入或更新metadata）
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
        @ meta data bytes: each meta data seperated by \x01,
                            name and value seperated by \x02
    # response body: none

**STORAGE_PROTO_CMD_DOWNLOAD_FILE**

    # function: download/fetch file from storage server（从storage server下载文件）
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body:
        @ file content

**STORAGE_PROTO_CMD_GET_METADATA**

    # function: get metat data from storage server（从storage server获取metadata）
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body
        @ meta data buff, each meta data seperated by \x01, name and value seperated by \x02

**STORAGE_PROTO_CMD_QUERY_FILE_INFO**

    # function: query file info from storage server
    # request body:
        @ FDFS_GROUP_NAME_MAX_LEN bytes: group name
        @ filename bytes: filename
    # response body:
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file size
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file create timestamp
        @ TRACKER_PROTO_PKG_LEN_SIZE bytes: file CRC32 signature

## php client端接口调用
**获取客户端版本号**

     string fastdfs_client_version()

**获取最近一个错误号**

    long fastdfs_get_last_error_no()

**获取最近一次错误的信息**

    string fastdfs_get_last_error_info()

**为http下载获取防盗连接**

    string fastdfs_http_gen_token(string remote_filename, int timestamp)
    生成防盗连接
    参数：
        @remote_filename:  远程文件名（不包含group name）
        @timestamp: 时间戳
    返回：成功则返回token串，否则返回false

**获取文件信息**

    array fastdfs_get_file_info(string group_name, string remote_filename)
    获取文件的meta info
    参数：
        @group_name: 文件所在的组名
        @remote_filename: 文件在storage server上的名字
    返回：
        成功则返回文件信息数组，否则返回false
        文件信息数组包括以下信息：
        create_timestamp: 文件创建的时间
        file_size: 文件大小
        source_ip_addr: 文件上传的源storage ip地址

    array fastdfs_get_file_info(string file_id)
    获取文件的meta info
    参数：
        @file_id: 组名+远程文件名

**判断文件是否存在**

    boolean fastdfs_storage_file_exist(string group_name, string remote_filename[, array tracker_server, array storage_server])
    判断文件是否存在
    参数：
        @group_name: 文件所在的组名
        @remote_filename: 远程文件名
        @array tracker_server: tracker_server数组，包含ip_addr, port and sock
        @array storage_server: storage_server数组，包含ip_addr, port and sock
    返回：
        存在返回true，否则返回false

    boolean fastdfs_storage_file_exist1(string file_id)

**上传文件**

    array fastdfs_storage_upload_by_filename(string local_filename[, string file_ext_name, array meta_list, string group_name, array tracker_server, array storage_server])
    参数：
        @local_filename: 本地要上传的文件名
        @file_ext_name: 文件扩展名，不包含dot
        @meta_list: 文件的一些元信息例如 array('width'=>1024,'height'=>768)
        @group_name: 指定文件存储到的组
        @array tracker_server: tracker_server数组，包含ip_addr, port and sock
        @array storage_server: storage_server数组，包含ip_addr, port and sock
    返回：
        成功返回数组，否则返回false
        数组包含以下字段：group_name, filename

    array fastdfs_storage_upload_by_filename1(string local_filename[, string file_ext_name, array meta_list, string group_name, array tracker_server, array storage_server])
    返回：
        成功则返回file_id, 否则返回false
        
    array fastdfs_storage_upload_by_filebuff(string file_buff[, string file_ext_name, array meta_list, string group_name, array tracker_server, array storage_server])
    参数：
        @file_buff: 文件内容
        @file_ext_name: 文件扩展名，不包含dot
        @meta_list: 文件的一些元信息例如 array('width'=>1024,'height'=>768)
        @group_name: 指定文件存储到的组
        @array tracker_server: tracker_server数组，包含ip_addr, port and sock
        @array storage_server: storage_server数组，包含ip_addr, port and sock
    返回：
        成功返回数组，否则返回false
        数组包含以下字段：group_name, filename
        
    array fastdfs_storage_upload_by_filebuff1(string file_buff[, string file_ext_name, array meta_list, string group_name, array tracker_server, array storage_server])
    
    array fastdfs_storage_upload_by_callback()
    
**append modify接口暂时不用**

**删除文件**

    boolean fastdfs_storage_delete_file(string group_name, string remote_filename[, array tracker_server, array storage_server])
    从storage server 删除文件
    参数：
        @group_name: 文件的组名
        @remote_filename: 文件在storage server上的名字
        @array tracker_server: tracker_server数组，包含ip_addr, port and sock
        @array storage_server: storage_server数组，包含ip_addr, port and sock
    返回：
        成功则返回true, 否则返回false
    
     boolean fastdfs_storage_delete_file1(string file_id[, array tracker_server, array storage_server])

**下载文件**

    string fastdfs_storage_download_file_to_buff(string group_name, string remote_filename[, long file_offset, long download_bytes, array tracker_server, array storage_server])
    从storage server下载文件
    参数：
        @group_name；文件的组名
        @remote_filename: 文件在storage server上的名字
        @file_offset:0(默认)
        @array tracker_server: tracker_server数组，包含ip_addr, port and sock
        @array storage_server: storage_server数组，包含ip_addr, port and sock
    返回：
        成功则返回文件的内容，否则返回false
        
    string fastdfs_storage_download_file_to_buff1(string file_id[, long file_offset, long download_bytes, array tracker_server, array storage_server])

    boolean fastdfs_storage_download_file_to_file(string group_name, string remote_filename, string local_filename[, long file_offset, long download_bytes, array tracker_server, array storage_server])
    从storage现在文件到本地文件
    参数：
        @local_filename: 要保存到本地的文件名
        其余参数同上

**设置、获取文件meta_data**

    boolean fastdfs_storage_set_metadata(string group_name, string remote_filename, array meta_list[, string op_type, array tracker_server, array storage server]
    设置文件metadata
    参数：
        @meta_list:meta信息数组：array('width'=>1024,'height'=>768)
        @op_type: 操作有两种，合并或者覆盖
            FDFS_STORAGE_SET_METADATA_FLAG_MERGE
            FDFS_STORAGE_SET_METADATA_FLAG_OVERWRITE
    返回：
        成功返回true,否则返回false
        
    array fastdfs_storage_get_metadata(string group_name, string remote_filename[, array tracker_server, array storage_server])
    获取文件的metadata
    返回：
        成功则返回meta_data数组，否则返回false
        
**连接/断开连接到server**

    array fastdfs_connect_server(string ip_addr, int port)
    连接到server
    参数：
        @ip_addr:  server的ip地址
        @port: server的端口
    返回：
        连接成功返回true;否则返回false
    
    array fastdfs_disconnect_server(string ip_addr, int port)
    断开到server的连接
    参数：
        同上

**Active 测试**

    boolean fastdfs_acitve_test(array server_info)
    给server发送active_test命令
    参数：
        @server_info: array: ip_addr, port, sock
    返回：
        active返回true，否则返回false

**获取连接**

    array fastdfs_tracker_get_connection()
    获取到tracker server的连接
    返回：
        成功则返回数组，否则返回false
        数组包括：ip_addr, port, sock
        
    array fastdfs_tracker_make_all_connection()
    连接到所有的tracker server
    返回：
        成功返回true;否则返回false

**列出tracker的组**
    
    array fastdfs_tracker_list_groups([string group_name, array tracker_server])
    获取组信息
    参数：
        @gourp_name:指定组名，默认列出所有组
    返回：
        组信息数组，数组每一项是一个组


**请求获取storage server to store（上传文件）**

    array fastdfs_tracker_query_storage_store([string group_name, array tracker_server])
    获取storage server的信息用以上传文件
    返回：
        成功返回一个storage server信息，保存在数组，否则返回false；
        数组包含：ip_addr, port, sock, store_path_index
        
    array fastdfs_tracker_query_storage_store_list
    同上，但返回的是storage server列表
    
**请求storage server to update（更新meta data）**

    array fastdfs_tracker_query_storage_update(string group_name, string remote_filename[, array tracekr_server]
    
**请求storage server to fetch（下载文件）**

    array fastdfs_tracker_query_storage_fetch(string gourp_name, string remote_filename[, array tracker_server])

**获取可下载某文件的storage server

    array fastdfs_tracker_query_storage_list(string group_name, string remote_filename[, array tracker_server])
    获取可以下载文件内容的storage server list
   
**删除storage**

    boolean fastdfs_tracker_delete_storage(string group, string storage_ip)
    将storage 从集群中删除
    参数：
        @group_name: storage所在的组名
        @storage_ip: storage server 的ip
    返回：
        成功返回true,否则返回false
