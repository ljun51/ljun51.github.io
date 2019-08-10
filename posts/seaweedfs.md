---
layout: post
title: SeaweedFS笔记
description: 记录学习SeaweedFS的过程
date: 2019-08-06
---

## 介绍
SeaweedFS是一个简单高扩展性的分布式文件系统，它可以：
1. 存储数十亿文件
2. 访问文件非常快

SeaweedFS作为一个Object Store而设计的系统可以高效的处理海量小文件。而不是采用以Master节点管理所有文件元数据的方式，Master节点仅仅管理文件Volume，Volume管理文件和文件元数据。这样就减轻了Master节点的压力，并将文件元数据转发到Volume服务，从而使访问文件的速度更快（只需要读取磁盘一次）。

每个文件的元数据只有40bytes的磁盘开销。使用O(1)的磁盘读取非常简单，可以使用实际用例测试性能。

SeaweedFS采用Facebook的Haystack设计论文实现，使用Facebook的Warm BLOB存储系统（f4）的实现了擦除编码。

SeaweedFS只使用Object Store就可以很好的运行，也可以使用Filer以支持目录和POSIX访问。Filer是一个独立、线性扩展、无状态的服务器，可以自定义元数据存储，如MySql、Postgres、Redis、Cassandra、LevelDB。

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

### 安装SeaweedFS

前往GitHub[下载][1]合适的版本，使用`weed -h`查看帮助说明：
> ./weed -h

### 启动Master Server

```
    ./weed master -h # 查看可用参数
    ./weed master
```

Master服务仪表盘可以在浏览器中输入`http://hostname:port`的形式访问，比如：http://localhost:9333。

Volume服务可以通过`http://hostname:port/ui/index.html`，比如：http://localhost:8080/ui/index.html。

### 启动Volume Servers

```
    ./weed volume -h # 查看可以参数
    ./weed volume -dir="/tmp/data1" -max=5 -mserver="localhost:9333" -port=8080 &
    ./weed volume -dir="/tmp/data2" -max=10 -mserver="localhost:9333" -port=8081 &
```

### 启动一个Master服务和一个Volume服务
> ./weed server -master.port=9333 -volume.port=8080 -dir="./data"

### 测试SeaweedFS

```
    ./weed upload -dir="/some/big/folder"
    ./weed upload -dir="/some/big/folder" -include=*.txt
```

`-dir`指定需要上传的目录，`-include`指定上传的文件

### 写文件

1. 发送HTTP POST、PUT或GET请求到`/dir/assign`获取`fid`和Volume服务URL：

```
    curl http://localhost:9333/dir/assign
    {"fid":"2,01fb400e5a","url":"127.0.0.1:8080","publicUrl":"127.0.0.1:8080","count":1}
```

2. 存储文件内容，根据响应URL及fid发送HTTP multi-part POST请求到`url + '/' + fid`:

```
    curl -F file=@/Users/lijun/Downloads/linux.png http://127.0.0.1:8080/2,01fb400e5a
    {"name":"linux.png","size":127093,"eTag":"28d861cf"}
```

如果需要更新文件内容，发送另一个POST请求即可。对于删除，发送HTTP DELETE请求：
> curl -X DELETE http://127.0.0.1:8080/2,01fb400e5a

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

```
    000: 没有复制，只有一个副本
    001: 在同一个机架上复制一次
    010: 在同一数据中心的不同机架上复制一次
    100: 在不同的数据中心复制一次
    200: 在另外两个不同的数据中心复制两次
    110: 在不同的机架上复制一次，在不同的数据中心复制一次
```

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
> http://localhost:9333/dir/assign?dataCenter=dc1

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

## API

### Master Server API

你可以在下面的HTTP API后面追加&pretty=y格式化输出JSON数据。

#### 分配file key

这些操作都是高性能的，只是在内存中增加一串数字：

```
    # 基本用法
    curl http://localhost:9333/dir/assign
    {"fid":"1,023f9685da","url":"127.0.0.1:8081","publicUrl":"127.0.0.1:8081","count":1}
    # 用指定的副本类型分配
    curl "http://localhost:9333/dir/assign?replication=001"
    # 指定预分配多少个文件ids
    curl "http://localhost:9333/dir/assign?count=5"
    # 在指定的数据中心分配
    curl "http://localhost:9333/dir/assign?dataCenter=dc1"
```

#### 查找volume

```
    curl "http://localhost:9333/dir/lookup?volumeId=1&pretty=y"
    {
    "volumeId": "1",
    "locations": [
        {
        "url": "127.0.0.1:8081",
        "publicUrl": "127.0.0.1:8081"
        }
    ]
    }

    # 使用file id查找
    curl "http://localhost:9333/dir/lookup?volumeId=1,023f9685da"
    # 如果已知collection，指定之后会更快速
    curl "http://localhost:9333/dir/lookup?volumeId=1&collection=turbo"
```

#### 预分配volumes

一个Volume一次只能写一个，如果需要增加并发性，可以预先分配大量Volume。下面是一些例子：

```
    # 指定副本策略
    curl "http://localhost:9333/vol/grow?replication=000&count=4"
    {"count":4}

    # 指定一个集合
    curl "http://localhost:9333/vol/grow?collection=turbo&count=4"

    # 指定数据中心
    curl "http://localhost:9333/vol/grow?dataCenter=dc&count=4"

    # 指定ttl
    curl "http://localhost:9333/vol/grow?ttl=5&count=4"
```

#### 删除collection

```
    # 删除集合
    curl "http://localhost:9333/col/delete?colletion=benchmark&pretty=y"
```

#### 检查系统状态

```
    curl "http://localhost:9333/cluster/status?pretty=y"
    {
        "IsLeader": true,
        "Leader": "localhost:9333"
    }

    curl "http://localhost:9333/dir/status?pretty=y"
    {
    "Topology": {
        "DataCenters": [
        {
            "Free": 0,
            "Id": "DefaultDataCenter",
            "Max": 15,
            "Racks": [
            {
                "DataNodes": [
                {
                    "EcShards": 0,
                    "Free": 0,
                    "Max": 10,
                    "PublicUrl": "127.0.0.1:8081",
                    "Url": "127.0.0.1:8081",
                    "Volumes": 10
                },
                {
                    "EcShards": 0,
                    "Free": 0,
                    "Max": 5,
                    "PublicUrl": "127.0.0.1:8080",
                    "Url": "127.0.0.1:8080",
                    "Volumes": 5
                }
                ],
                "Free": 0,
                "Id": "DefaultRack",
                "Max": 15
            }
            ]
        }
        ],
        "Free": 0,
        "Max": 15,
        "layouts": [
        {
            "collection": "",
            "replication": "000",
            "ttl": "",
            "writables": [
            1,
            2,
            3,
            4,
            5,
            6,
            7,
            8,
            9,
            10,
            11
            ]
        },
        {
            "collection": "",
            "replication": "001",
            "ttl": "",
            "writables": null
        },
        {
            "collection": "turbo",
            "replication": "000",
            "ttl": "",
            "writables": [
            12,
            13,
            14,
            15
            ]
        }
        ]
    },
    "Version": "30GB 1.42"
    }
```

### Volume Server API

你可以在下面的HTTP API后面追加&pretty=y格式化输出JSON数据。

#### 上传文件

```
    curl -F file=@/Users/lijun/Downloads/linux.png http://127.0.0.1:8080/2,01fb400e5a
    {"name":"linux.png","size":127093,"eTag":"28d861cf"}
```
返回的大小是存储在SeaweedFS上的大小，有时文件会根据mime类型自动进行gzip压缩。

#### 直接上传文件

```
    curl -F file=@/Users/lijun/Downloads/linux.png http://localhost:9333/submit
    {"eTag":"28d861cf","fid":"11,0380f00042","fileName":"linux.png","fileUrl":"127.0.0.1:8081/11,0380f00042","size":127093}
```
主服务器将获得文件ID并将文件存储到正确的卷服务器。它是一个方便的API，在分配文件ID时不支持不同的参数。（或者您可以添加支持并发送推送请求。）

#### 删除文件

```
    curl -X DELETE http://127.0.0.1:8081/11,0380f00042
```

#### 查看分块大文件的文件内容

```
    curl http://127.0.0.1:8081/11,0380f00042?cm=false
```

#### 检查卷服务状态

```
    curl "http://127.0.0.1:8080/status?pretty=y"
    {
    "Version": "30GB 1.42",
    "Volumes": [
        {
        "Id": 2,
        "Size": 127138,
        "ReplicaPlacement": {
            "SameRackCount": 0,
            "DiffRackCount": 0,
            "DiffDataCenterCount": 0
        },
        "Ttl": {
            "Count": 0,
            "Unit": 0
        },
        "Collection": "",
        "Version": 3,
        "FileCount": 1,
        "DeleteCount": 0,
        "DeletedByteCount": 0,
        "ReadOnly": false,
        "CompactRevision": 0,
        "ModifiedAtSecond": 0
        },
        {
        "Id": 3,
        "Size": 0,
        "ReplicaPlacement": {
            "SameRackCount": 0,
            "DiffRackCount": 0,
            "DiffDataCenterCount": 0
        },
        "Ttl": {
            "Count": 0,
            "Unit": 0
        },
        "Collection": "",
        "Version": 3,
        "FileCount": 0,
        "DeleteCount": 0,
        "DeletedByteCount": 0,
        "ReadOnly": false,
        "CompactRevision": 0,
        "ModifiedAtSecond": 0
        },
        ...
    }
```

### Filer Server API

你可以在下面的HTTP API后面追加&pretty=y格式化输出JSON数据。

#### Filer server

```
    # POST/PUT/GET文件
    # 创建或覆盖文件，目录/path/to会自动创建
    POST /path/to/file
    # 获取文件内容
    GET /path/to/file
    # 创建或覆盖文件，使用multipart请求中的文件名
    POST /path/to/
    # 返回JSON格式的子目录及文件列表
    GET /path/to/
    Accept: application/json
```

示例：
```
    curl -F file=@seaweedfs_darwin_amd64.tar "http://localhost:8888/seaweedfs/"
    {"name":"seaweedfs_darwin_amd64.tar"}
    
    curl -F file=@seaweedfsintroduction-190411215802.pdf "http://localhost:8888/seaweedfs/seaweedfsintroduction.pdf"
    {"name":"seaweedfsintroduction-190411215802.pdf","size":547712,"fid":"8,0a4857e728","url":"http://127.0.0.1:8081/8,0a4857e728"}

    curl -H "Accept: application/json" "http://localhost:8888/seaweedfs/?pretty=y"
    {
      "Path": "/seaweedfs",
      "Entries": [
        {
          "FullPath": "/seaweedfs/seaweedfs_darwin_amd64.tar",
          "Mtime": "2019-08-06T09:49:52+08:00",
          "Crtime": "2019-08-06T09:49:52+08:00",
          "Mode": 432,
          "Uid": 501,
          "Gid": 20,
          "Mime": "",
          "Replication": "000",
          "Collection": "",
          "TtlSec": 0,
          "UserName": "",
          "GroupNames": null,
          "SymlinkTarget": "",
          "chunks": [
            {
              "file_id": "10,08b1b7f9be",
              "size": 33554432,
              "mtime": 1565056192271756000,
              "fid": {
                "volume_id": 10,
                "file_key": 8,
                "cookie": 2981624254
              }
            },
            {
              "file_id": "3,094e97c6a7",
              "offset": 33554432,
              "size": 17553408,
              "mtime": 1565056192485648000,
              "fid": {
                "volume_id": 3,
                "file_key": 9,
                "cookie": 1318569639
              }
            }
          ]
        },
        {
          "FullPath": "/seaweedfs/seaweedfsintroduction.pdf",
          "Mtime": "2019-08-06T10:16:17+08:00",
          "Crtime": "2019-08-06T10:16:17+08:00",
          "Mode": 432,
          "Uid": 501,
          "Gid": 20,
          "Mime": "application/pdf",
          "Replication": "000",
          "Collection": "",
          "TtlSec": 0,
          "UserName": "",
          "GroupNames": null,
          "SymlinkTarget": "",
          "chunks": [
            {
              "file_id": "8,0a4857e728",
              "size": 547712,
              "mtime": 1565057777513573000,
              "e_tag": "\"2f6a68fa\"",
              "fid": {
                "volume_id": 8,
                "file_key": 10,
                "cookie": 1213720360
              }
            }
          ]
        }
      ],
      "Limit": 100,
      "LastFileName": "seaweedfsintroduction.pdf",
      "ShouldDisplayLoadMore": false
    }
```

#### 删除文件

> curl -X DELETE http://localhost:8888/path/to/file

#### 删除文件夹

> curl -X DELETE http://localhost:8888/path/to/dir?recursive=true

## Replication

SeaweedFS是支持副本的。SeaweedFS实现副本不是在文件级别，而是在卷级别。

### 如何使用

1. 启动weed master，可选的指定副本的默认级别

    ./weed master -defaultReplication=001

2. 启动weed volume

    ./weed volume -port=8080 -dir=/tmp/1 -max=100 -mserver="master_address:9333" -dataCenter=dc1 -rack=rack1
    ./weed volume -port=8081 -dir=/tmp/1 -max=100 -mserver="master_address:9333" -dataCenter=dc1 -rack=rack1

在另一个机架：

    ./weed volume -port=8080 -dir=/tmp/1 -max=100 -mserver="master_address:9333" -dataCenter=dc1 -rack=rack2
    ./weed volume -port=8081 -dir=/tmp/2 -max=100 -mserver="master_address:9333" -dataCenter=dc1 -rack=rack2

### 副本类型

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

### 指定的数据中心分配File Key

    http://localhost:9333/dir/assign?dataCenter=dc1

### 使用TTL存储文件

SeaweedFS使用key~file的存储方式，文件可以指定Time To Live(TTL).

#### 如何使用

如果想指定一个文件的TTL为3min。首先，分配一个TTL为3min的fileId：

```
    curl http://localhost:9333/dir/assign?ttl=3m
    {"fid":"18,018a03d05e","url":"127.0.0.1:8081","publicUrl":"127.0.0.1:8081","count":1}
```

第二，使用fileId存储文件到卷服务器

    curl -F "file=@linux.png" http://127.0.0.1:8081/18,018a03d05e?ttl=3m

文件写完后，如果在TTL到期之前读取，文件内容将照常返回。但是如果在TTL到期后读取，则该文件将被报告为丢失并返回未找到的http响应状态。

对于ttl=3m的下一次写入，将使用ttl=3m的相同卷，但是：
1. 如果ttl=3m的卷已经写满，新卷将会被创建。
2. 如果3分钟都没有写入操作，这些卷将会被删除。

你或许已经注意到了，ttl=3m使用了2次，第一次分配文件ID，第二次上传实际的文件。这两个TTL没有要求同时请求，只要volume TTL比file TTL大就没有问题。

这为调整文件TTL提供了一定的灵活性，同时减少了TTL卷的数量变化，从而简化TTL卷的管理。

#### 支持的TTL格式

TTL的格式为一个整数后跟一个单位，单位可以是'm', 'h', 'w', 'M', 'y'.
* 3m: 3 minutes
* 4h: 4 hours
* 5d: 5 days
* 6w: 6 weeks
* 7M: 7 months
* 8y: 8 years

#### 如何保证高效

TTL似乎很容易实现，因为如果时间超过TTL，我们只需要将文件报告为缺失。然而，真正的困难是从过期文件中有效地回收磁盘空间，类似于JVM内存垃圾收集，这是一项复杂的工作，需要多年的努力。

Memcached也支持TTL。它通过将条目放入固定大小的slab来解决这个问题。如果一块slab过期，则不需要任何工作，并且可以立即覆盖slab。但是，这种固定大小的slab方法不适用于文件，因为文件内容很少精确地适合于slab。

SeaweedFS非常简单有效地解决了这个磁盘空间垃圾收集问题。与常规系统实现的主要区别之一是TTL与卷以及每个特定文件相关联。在文件ID分配步骤期间，文件ID将分配给具有匹配TTL的卷。定期检查卷（默认情况下每隔5~10秒）。如果已达到最新到期时间，则整个卷中的所有文件都将过期，并且可以安全地删除卷。

#### 实现详情

1. 当分配file key时，master挑选匹配TTL的TTL卷。如果这样的卷不存在，在创建一些。
2. Volume服务器写入文件内容时会带上过期时间。当文件被访问时，如果文件过期则返回文件不存在。
3. Volume服务器追踪每个卷的最大过期时间，已经过期的卷不再报告给master。
4. Master服务器对于已经过期不再报告的卷，不在分配写请求。
5. 在大约10%的TTL时间或最多10分钟后，Volume服务器将会删除已过期的卷。

#### 部署

对于生产环境，要考虑TTL卷的最大大小。如果写入频繁，TTL卷将会增长到最大的卷大小；相反，如果写入不是很频繁的时候，频繁的删除卷就会浪费资源。可以考虑减少最大卷大小。

推荐不要在同一个集群混用TTL卷和非TTL卷。因为最大卷的默认大小是30GB，而且这个配置是集群级别的。

### Master Server容错

有些用户不需要SPOF（单点故障），但是有没有SPOF似乎成为架构师选择解决方案的标准，虽然goole多年来一直使用单一Master运行其文件系统。SeaweedFS支持容错不是太难。

#### 启动多个服务

下面的命令启动3个master server、3个volume server：

```
    weed server -master.port=9333 -dir=./1 -volume.port=8080 -master.peers=localhost:9333,localhost:9334,localhost:9335
    weed server -master.port=9334 -dir=./2 -volume.port=8081 -master.peers=localhost:9333,localhost:9334,localhost:9335
    weed server -master.port=9335 -dir=./3 -volume.port=8082 -master.peers=localhost:9333,localhost:9334,localhost:9335
```

#### 如何工作

Master服务之间使用Raft协议保持一致性、选举领导者。领导者接管管理卷的所有工作，分配文件ID，其他所有master只是简单的转发请求给领导者。

如果领导着停止服务，另一个领导者会被选举，所有卷服务发送心跳和卷信息给新的领导者，新领导者承担所有责任。

在新老领导着的转换期间，新领导者可能只会有部分卷服务器的信息，这就意味着那行还没有发送心跳给新领导者的卷暂时不可用。

#### Master标识

当指定master节点时，节点之间使用户hostname、IP和端口标识身份。hostname或IP在启动master服务之前可以通过“-ip”指定。

下面分别启动master和volume服务，一般可以启动3或5个master服务，然后启动volume服务：

```
    weed master -port=9333 -mdir=./1 -peers=localhost:9333,localhost:9334,localhost:9335
    weed master -port=9334 -mdir=./2 -peers=localhost:9333,localhost:9334,localhost:9335
    weed master -port=9335 -mdir=./3 -peers=localhost:9333,localhost:9334,localhost:9335
    # 启动volume服务，指定任意一个master
    weed volume -dir=./1 -port=8080 -mserver=localhost:9333,localhost:9334,localhost:9335
    weed volume -dir=./2 -port=8081 -mserver=localhost:9333,localhost:9334,localhost:9335
    weed volume -dir=./3 -port=8082 -mserver=localhost:9333,localhost:9334,localhost:9335
    # 推荐使用所有master服务地址，但不强制，也可以指定部分master服务地址
    weed volume -dir=./3 -port=8082 -mserver=localhost:9333
    weed volume -dir=./3 -port=8082 -mserver=localhost:9333,localhost:9334
```

这里的6条命令实际和前面的3个命令的效果是一样的。当有多个master服务，其中一个服务停止的时候，卷服务器可以切换到其他节点。

#### FAQ

Q：在正在运行中的SeaweedFS集群可以添加另外的master服务节点吗？
A：和zookeeper一样，节点一般不会经常改变。但是如果想添加新的master服务节点，需要停止运行中的集群，在启动是重新指定master节点。对于volume服务，可以使用现有的master服务节点。

## Filer

### Filer安装

请将`filer.toml`文件增加到当前目录或$HOME/.seaweedfs/或/etc/seaweedfs/，可以通过下面的方式生产file.toml:

> weed scaffold -config=filer -output="."

或者只是查看：

> weed scaffold -config=filer

### 目录和文件

当我们谈论文件系统系统的时候，许多人第一个想到的是目录、目录下的文件。我们期望SeaweedFS与linux的FUSE、Hadoop的文件管理方式类似。

#### 使用示例

首先运行`weed scaffold -config=filer`查看`file.toml`示例文件。最简单的filer.toml是：

```
    [leveldb]
    enabled = true
    dir = "."
```

两种方式启动weed filer：

```
    # 假设已经启动了weed master和volume
    weed filer
    # 假设没有启动weed master, volume
    # 这个命令启动master server、volume server、filer，这和单独启动每个服务是相同的。
    weed server filer=true
```

现在你可以添加或删除文件，甚至浏览子目录：
```
    # POST写入一个文件并读取
    curl -F "filename=@README.md" "http://localhost:8888/path/to/sources/"
    curl "http://localhost:8888/path/to/sources/README.md"
    
    # POST重命名一个文件并读取
    curl -F "filename=@Makefile" "http://localhost:8888/path/to/sources/new_name"
    curl "http://localhost:8888/path/to/sources/new_name"

    # 列出子目录及文件
    visit "http://localhost:8888/path/to/sources/"

    # 如果在目录下有大量的文件，通过下面的方式分页
    visit "http://localhost:8888/path/to/sources/?lastFileName=abc.txt&limit=50"
```

#### 架构

Filer有一个连接到Master的持久客户端，以获取所有卷的位置更新。不需要通过网络往返来查找卷ID的位置。

对于文件读取：
1. Filer查找Filer Store的元数据，Filer Store可以是Cassandra、Mysql、Postgres、Redis、LevelDB。
2. Filer从卷服务器读取并传递给读取请求。
![SeaweedFS Filer架构][2]

对于文件写入：
1. 客户端流文件到Filer。
2. Filer上传数据到Weed Volume服务器，并将大文件分块。
3. Filer将元数据及文件块写入到Filer Store。

### 文件命令操作

`weed filer.copy`可以拷贝文件、文件列表或目录到Filer。

```
    # 拷贝当前目录所有go文件到filer的github目录，目录结构也会被拷贝
    weed filer.copy -include *.go . http://localhost:8888/github/
```

`weed copy`是非常高效的。它通过Master服务器分配fileId，提交文件内容到Volume服务器，然后在Filer中创建一条记录。如果是大文件将会自动拆分为文件块。

### Mount

#### 支持的功能

使用`weed mount`可以像操作本地文件一样操作Filer，支持下面的操作：
* file read/write
* create new file
* mkdir
* list
* remove
* rename
* chmod
* chown
* soft link
* display free disk space

## 使用案例

### 保存不通大小的图片

每个图片在数据库中通常保存一个file key。但是，有时候需要保存多个版本，比如：缩略图、小尺寸、中尺寸、大尺寸、原始图。SeaweedFS可以使用相同的file key存储不同的版本，解决办法是先分配5个file key：
```
    curl http://<host>:<port>/dir/assign?count=5
    {"fid":"22,0105b64621","url":"127.0.0.1:8080","publicUrl":"127.0.0.1:8080","count":5}
```

在卷服务器上保存图片的5个版本，每个图片的URL可以是：
```
    curl -F file=@/upload.png http://127.0.0.1:8080/22,0105b64621
    curl -F file=@/upload.png http://127.0.0.1:8080/22,0105b64621_1
    curl -F file=@/upload.png http://127.0.0.1:8080/22,0105b64621_2
    curl -F file=@/upload.png http://127.0.0.1:8080/22,0105b64621_3
    curl -F file=@/upload.png http://127.0.0.1:8080/22,0105b64621_4
```

### 覆盖mine types

正确的做法是：
> curl -F "file=@myImage.png;type=image/png" http://127.0.0.1:8080/5,2730a7f18b44

错误的做法是：
> curl -H "Content-Type:image/png" -F file=@myImage.png http://127.0.0.1:8080/5,2730a7f18b44

### 安全SeaweedFS

简单的方式是使用防火墙支持所以的master服务器和volume服务器。

白名单机制也是支持的，只有白名单中的IP列表才有写入权限：
```
    weed master -whiteList="::1,127.0.0.1"
    weed volume -whiteList="::1,127.0.0.1"
```

### 数据迁移

```
    weed master -mdir="/tmp/mdata" -defaultReplication="001" -ip="localhost" -port=9334
    weed volume -dir=/tmp/vol1/ -mserver="localhost:9334" -ip="localhost" -port=8081
    weed volume -dir=/tmp/vol2/ -mserver="localhost:9334" -ip="localhost" -port=8082
    weed volume -dir=/tmp/vol3/ -mserver="localhost:9334" -ip="localhost" -port=8083
```

```
    ls vol1 vol2 vol3
    vol1:
    1.dat 1.idx 2.dat 2.idx 3.dat 3.idx 5.dat 5.idx
    vol2:
    2.dat 2.idx 3.dat 3.idx 4.dat 4.idx 6.dat 6.idx
    vol3:
    1.dat 1.idx 4.dat 4.idx 5.dat 5.idx 6.dat 6.idx
```

停止所以master、volume服务，将vol3下的文件移到vol1和vol2。移动x.dat和x.idx从一个volume服务到另一个volume服务是没问题的，因为它们完全一致，可以使用md5校验。

```
    md5 vol1/1.dat vol2/1.dat
    MD5 (vol1/1.dat) = c1a49a0ee550b44fef9f8ae9e55215c7
    MD5 (vol2/1.dat) = c1a49a0ee550b44fef9f8ae9e55215c7
    md5 vol1/1.idx vol2/1.idx
    MD5 (vol1/1.idx) = b9edc95795dfb3b0f9063c9cc9ba8095
    MD5 (vol2/1.idx) = b9edc95795dfb3b0f9063c9cc9ba8095
```

```
    ls vol1 vol2 vol3
    vol1:
    1.dat 1.idx 2.dat 2.idx 3.dat 3.idx 4.dat 4.idx 5.dat 5.idx 6.dat 6.idx
    vol2:
    1.dat 1.idx 2.dat 2.idx 3.dat 3.idx 4.dat 4.idx 5.dat 5.idx 6.dat 6.idx
    vol3:
```

```
    weed master -mdir="/tmp/mdata" -defaultReplication="001" -ip="localhost" -port=9334
    weed volume -dir=/tmp/vol1/ -mserver="localhost:9334" -ip="localhost" -port=8081
    weed volume -dir=/tmp/vol2/ -mserver="localhost:9334" -ip="localhost" -port=8082
```

现在完成了从locahost:8083到localhost:8081/localhost:8082的数据迁移。

## 运维

### 系统监控

SeaweedFS使用Prometheus存储监控数据，使用Grafana可视化展示。SeaweedFS向Prometheus Push Gateway发布指标，网关传递给Prometheus服务器。

#### 配置

只需要将监控地址加到`weed master`或`weed server`命令行后面，如果有多个master，分别追加命令行参数。
```
    weed master -metrics.address=<prometheus_gateway_host_name>:<prometheus_gateway_port>
    # example
    weed master -metrics.address=localhost:9091

    weed server -metrics.address=<prometheus_gateway_host_name>:<prometheus_gateway_port>
    # example
    weed server -metrics.address=localhost:9091
```

SeaweedFS filer管理器或volume服务器将从master服务器读取此度量标准配置，并将度量标准直接报告给Prometheus网关。

### weed shell

`weed shell`启动一个交互式的控制台可以处理一些运维工作。下面的操作是支持的：

对于master、volume服务器：
* 列出所有集合分类
* 列出所有卷
* 修复正在复制的卷

对于文件管理器：
* 显示磁盘使用量

例如：
```
    $ weed shell
    > fs.du http://localhost:8888/seaweedfs/
    block:   3	byte:  51655552	/seaweedfs
```

有时候一些volume服务下线、新的volume服务器被加入，可以运行下面的命令修复正在被复制的卷：
```
    # check any volume that are under replicated, and there are servers that meet the replica placement requirement
    $ echo "volume.fix.replication -n " | weed shell
    replicating volume 241 001 from localhost:8080 to dataNode 127.0.0.1:7823 ...

    # found one, let's really do it
    $ echo "volume.fix.replication" | weed shell
    replicating volume 241 001 from localhost:8080 to dataNode 127.0.0.1:7823 ...

    # all volumes are replicated now
    $ echo "volume.fix.replication -n" | weed shell
    no under replicated volumes
```

### 数据备份

> weed backup -server=master:port -dir=. -volumeId=5
 
上面的命令备份卷5，如果卷5不存在，本地不会创建文件。这样你就可以创建一个简单的脚本从1循环到100，所以存在的卷都会被创建，不存在的也不会受影响。

## 安全

### 安全概述

SeaweedFS是有多个volume服务器组成的分布式系统，volume服务器有没有授权就能访问系统的风险，如果想从任意地方访问volume服务，就需要考虑没人可以恶意篡改数据。

首先要解决volume服务器的问题，下面的内容还没有涵盖到：
1. master服务http REST服务化
2. filer服务http REST服务化

#### 生成`security.toml`文件

SeaweedFS服务通常支持两种操作：gRPC和REST。

#### 安全的gRPC操作

下面的操作已通过gRPC实现：
* 从filer到master的请求
* 从master到volume server的请求
* 从filer、其他客户端（mount、s3、filer.replicate等）到volume server的删除操作
* 从客户端到filer的请求

通过自定义security.toml文件，可以选择使用TLS保护所有gRPC操作。

#### 安全Volume Servers

除了上面提到的gRPC之外，只能通过文件上传、更新、删除操作来更改卷服务器。具体的每个文件操作控制可以使用Json Web Token（JWT）授权。

##### 基于JWT的访问控制

要启用基于JWT的访问控制：
1. 通过`weed scaffold -config=security`生成`security.toml`文件
2. 设置`jwt.signing.key`的加密字符串
3. 拷贝`security.toml`文件到其他master、volue服务器

##### 基于JWT的访问控制是如何工作的

* 对于上传一个新文件，当通过http://<master>:<port>/dir/assign请求file key时，master服务使用`jwt.signing.key`生成一个签名的JWT，并设置响应头`Authorization`，JWT的有效时间是10秒。
* 对于更新或删除一个文件，JWT可以从http://<master>:<port>/dir/lookup?fileId=xxxxx的响应头中读取`Authorization`。
* 当向volume服务器发送上传、更新、删除操作时，请求头`Authorization`应该是JWT字符串，当volume服务使用`jwt.signing.key`的JWT验证通过时，操作才会被允许。

JWT摘要：
* JWT被设置在`/dir/assign`或`/dir/lookup`的`Authorization`相应头
* JWT从请求头中读取`Authorization`
* JWT的有效时间是10秒
* JWT一次只能创建、修改、删除一个文件
* Volume服务的HTTP只能读取，每次只能一个文件，不能迭代读取所有文件
* 当企业jwt.signing时，将禁用其他卷服务器的HTTP访问

##### JWT的读取访问控制

卷服务器还可以检验JWT的读取，此模式不适用于weed filer。如果卷服务器暴露在公开环境，但是你又不希望任何人都可以访问（比如付费内容），这个功能将会很有用。

* 要启用这个功能，设置`security.toml`文件的`jwt.signing.read.key`
* 要获取JWT读取文件内容，可以取http://<master>:<port>/dir/lookup?fileId=xxxxx&read=yes的响应头`Authorization`。

前往[官方文档][1].

[1]: https://github.com/chrislusf/seaweedfs
[2]: http://ljun51.github.io/images/FilerRead.png
