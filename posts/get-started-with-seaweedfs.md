---
layout: post
title: SeaweedFS笔记
description: 记录学习SeaweedFS的过程
date: 2019-07-26
---

## 介绍
SeaweedFS是一个简单高扩展性的分布式文件系统，它可以：
1. 存储数十亿文件
2. 访问文件非常快

SeaweedFS作为一个Object Store而设计可以高效的处理小文件。而不是采用以Master节点管理所以文件元数据的方式，Master节点仅仅管理文件Volumes，Volumes管理文件和文件元数据。这样就减轻了Master节点的压力，并将文件元数据传播到Volume服务器，从而使访问文件的速度更快（只需要读取磁盘一次）。

每个文件的元数据只有40bytes的磁盘开销。使用O(1)的磁盘读取非常简单，可以使用实际用例测试性能。

SeaweedFS采用Facebook的Haystack设计论文实现，使用Facebook的Warm BLOB存储系统（f4）的实现了擦除编码。

SeaweedFS使用Object Store就可以很好的运行，也可以使用Filer以支持目录和POSIX访问。Filer是一个独立、线性扩展、无状态的服务器，可以自定义元数据存储，如MySql、Postgres、Redis、Cassandra、LevelDB。

## 特性

### 附加特性

* 可以选择无副本或不同的副本等级、机架和数据中心感知
* 自动Master服务故障转移，没有单点故障（SPOF）
* 自动Gzip压缩，Gzip压缩取决于文件的mime类型
* 在删除或更新后自动压缩以回收磁盘空间
* 同一个集群服务，节点可以有不同的磁盘空间、文件系统、操作系统
* 添加、删除服务节点不会导致数据re-balancing
* 可选的修复图片的方向
* 支持Etag、Accept-Range、Last-Modified等
* 支持in-memory/leveldb/boltdb/btree模式以实现内存于性能之间的平衡
* 支持重新平衡读、写Volumes

### Filer特性

* filer server: 通过http提供传统的目录、文件访问
* mount filer: 通过FUSE可以像访问本地目录一样直接读写文件
* Amazon S3 compatible API: 使用S3工具访问文件
* Erasure Coding for warm storage: Rack-Aware 10.4擦除编码可以减少存储成本并提高可用性
* Hadoop Compatible File System: 用于从Hadoop/Spark/Flink等作业访问文件
* Async Backup To Cloud: 极速本地访问及Amazon S3、Azure、Google Cloud Storage备份
* WebDAV: 作为Mac、Windows上的映射驱动器，或移动设备

## 使用示例

默认情况下，Master节点运行在9333端口，Volume节点运行在8080端口。让我们启动一个Master节点、两个Volume节点。理想情况下，这些节点应该运行在不同的机器上，现在使用localhost作为示例。

SeaweedFS使用HTTP REST操作读、写、删除，响应使用JSON或JSONP格式。

### 启动Master Server

    ./weed master

### 启动Volume Servers

    weed volume -dir="/tmp/data1" -max=5 -mserver="localhost:9333" -port=8080 &
    weed volume -dir="/tmp/data2" -max=10 -mserver="localhost:9333" -port=8081 &

### 写文件

1. 发送HTTP POST、PUT或GET请求到`/dir/assign`获取`fid`和Volume服务URL：

    curl http://localhost:9333/dir/assign
    {"fid":"2,01fb400e5a","url":"127.0.0.1:8080","publicUrl":"127.0.0.1:8080","count":1}

2. 存储文件内容，根据响应URL及fid发送HTTP multi-part POST请求到`url + '/' + fid`:

    curl -F file=@/Users/lijun/Downloads/linux.png http://127.0.0.1:8080/2,01fb400e5a
    {"name":"linux.png","size":127093,"eTag":"28d861cf"}

如果需要更新文件内容，发送另一个POST请求即可。对于删除，发送HTTP DELETE请求：

    curl -X DELETE http://127.0.0.1:8080/2,01fb400e5a

### 保存File Id

现在可以保存`fid`到数据库，在这个示例中是`2,01fb400e5a`。

开始的数字2是volume id，逗号之后01是file key, fb400e5a是file cookie。

volume id是一个无符号32bit整数，file key是无符号64bit整数，file cookie是无符号32bit整数，可以防止URL路径猜测。

file key、file cookie使用16进制编码，你可以使用自定义格式存储<volume id, file key, file cookie>元组，或者简单的将`fid`存储为字符串。

如果作为字符串存储，在理想情况下，需要8+1+16+8=33bytes。如果不是绰绰有余，char(33)就足够了，因为大多数用途不需要2^32卷。

如果真的需要考虑存储空间，你可以使用自己的存储格式。使用4byte整数存储volume id，8byte长整存储file key，4byte整数存储file cookie。16byte就足够了。

### 读取文件

使用volumeId查询volume服务器的URL：

    curl http://localhost:9333/dir/lookup?volumeId=2
    {"volumeId":"2","locations":[{"url":"127.0.0.1:8080","publicUrl":"127.0.0.1:8080"}]}

大部分情况下你可以缓存这个结果，因为通常不会有太多的volume服务器，volume也不会轻易移动。根据副本类型的不同，一个volume可能会有多个副本位置，只需要随机挑选一个即可。

现在您可以通过公共URL，或直接从卷服务器读取：
> http://localhost:8080/2,01fb400e5a.jpg

注意上面添加了一个文件扩展`.jpg`，这是可选的，只是客户端指定文件内容类型的一种方式。你可以使用下面中的任何一种：

    http://localhost:8080/2/01fb400e5a/my_preferred_name.jpg
    http://localhost:8080/2/01fb400e5a.jpg
    http://localhost:8080/2,01fb400e5a.jpg
    http://localhost:8080/2/01fb400e5a
    http://localhost:8080/2,01fb400e5a

如果想获取图像的缩放版本，添加下面的参数：

    http://localhost:8080/2/01fb400e5a.jpg?height=200&width=200
    http://localhost:8080/2/01fb400e5a.jpg?height=200&width=200&mode=fit
    http://localhost:8080/2/01fb400e5a.jpg?height=200&width=200&mode=fill

### Rack-Aware和Data Center-Aware复制

SeaweedFS在volume级别使用副本策略。当你获取file id时，可以指定副本策略：
> curl http://localhost:9333/dir/assign?replication=001

副本策略的可选参数可以为：

    000: 没有复制，只有一个副本
    001: 在同一个机架上复制一次
    010: 在同一数据中心的不同机架上复制一次
    100: 在不同的数据中心复制一次
    200: 在另外两个不同的数据中心复制两次
    110: 在不同的机架上复制一次，在不同的数据中心复制一次

所以假设副本类型是xyz

    x: 在其他的数据中心的副本数
    y: 在其他机架同一个数据中心的副本数
    z: 在其他服务器相同的机架上的副本数

x,y,z的可选值是0，1，2，所以有9中副本类型，可以很容易的扩展。每个副本类型在物理存储上将会创建x+y+z+1个volume数据文件。启动Master时可以指定默认的副本策略。

### 在指定的数据中心分配File Key

在启动volume服务时指定数据中心的名称：

    weed volume -dir=/tmp/data1 -port=8080 -dataCenter=dc1
    weed volume -dir=/tmp/data2 -port=8081 -dataCenter=dc2

当请求一个file key时，"dataCenter"参数限制在指定数据中心已分配volume上。例如，指定分配的volume在"dc1":

    http://localhost:9333/dir/assign?dataCenter=dc1

## 架构

通常，分布式文件系统将每个文件分成块，Master保持文件名、块索引到块句柄的映射，即每个块服务器具体的块。这种设计的瓶颈是Master节点不能高效的处理小文件，因为所有的请求需要通过Master，因此对于高并发用户来说可能无法很好地扩展。

SeaweedFS不管理块，而是管理数据卷。每个数据卷大小为32GB，可以容纳大量文件。每个存储节点可以有许多数据卷。因此主节点只需要存储有关卷的元数据，这是一个相当少量的数据，并且通常是稳定的。

实际文件元数据存储在卷服务器上的每个卷中。由于每个卷服务器仅管理其自己磁盘上的文件元数据，而且每个文件只有16个字节，因此所有文件访问可以从内存中读取文件元数据，并且只需要一个磁盘操作即可实际读取文件数据。

### Master Server和Volume Server

架构非常简单。实际数据存储在存储节点上的卷中。一个卷服务器可以有多个卷，并且都可以通过基本身份验证支持读写访问。所有卷都由主服务器管理。主服务器包含卷ID到卷服务器映射。这是相对静态的信息，可以做缓存。在每个写请求中，主服务器还生成一个文件Key，这是一个不断增长的64位无符号整数。由于写请求通常不像读请求那样频繁，因此一个主服务器应该能够很好地处理并发。

### 读写文件

当客户端发送写请求时，master服务返回(volume id, file key, file cookie, volume node url)，客户端通过这些信息POST文件内容到volume节点。

当客户端基于（volume id, file key, file cookie）读取文件，通过volume id或缓存获取URL，然后通过GET获取内容。

### 存储大小

在当前实现中，每个卷可以容纳32个gibibytes（32GiB或8x2^32个字节）。这是因为我们将内容与8个字节对齐。我们可以通过更改2行代码轻松地将其增加到64GiB或128GiB或更多，代价是由于对齐而导致一些浪费的填充空间。可以有4个gibibytes（4GiB或2^32字节）的卷。因此总系统大小为8 x 4GiB x 4GiB，即128 exbibytes（128EiB或2^67字节）。每个文件大小都限制为卷大小。

### 节省内存

存储在卷服务器上的所有文件元信息都可以从内存中读取而无需磁盘访问。每个文件只需要一个16字节的<64位密钥，32位偏移，32位大小>的映射。当然，每个映射都有自己的映射空间成本，但通常磁盘空间会在内存耗尽之前耗尽。
