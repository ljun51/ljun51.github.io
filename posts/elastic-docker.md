---
layout: post
title: Elastic Docker
description: Elastic栈Docker容器化
date: 2019-07-10
---

# Elastic栈Docker容器化

## 单节点安装

单节点Elastic栈常用于开发环境，通常必须的环境是Elasticsearch + Kibana，可选的包括Beats和Logstash等。

### 安装Elasticsearch

```
    docker pull docker.elastic.co/elasticsearch/elasticsearch:7.2.0
    docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.2.0
```

Elasticsearch执行上面的命令会输出启动信息，当然也可以加入`-d`参数使Elasticsearch在后台运行：
```
    docker run -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.2.0
```

### 安装Kibana

执行下面的命令之前需要查看Elasticsearch容器的ID或名称`YOUR_ELASTICSEARCH_CONTAINER_NAME_OR_ID`，可以通过`docker ps`查看容器的ID（使用`docker run -d`运行Elasticsearch也会返回容器ID）：

```
    docker ps
    CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                                            NAMES           970945a28453        docker.elastic.co/elasticsearch/elasticsearch:7.2.0   "/usr/local/bin/dock…"   13 seconds ago      Up 11 seconds       0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   determined_nobel
```

```
    docker pull docker.elastic.co/kibana/kibana:7.2.0
    # 格式如下
    docker run --link YOUR_ELASTICSEARCH_CONTAINER_NAME_OR_ID:elasticsearch -p 5601:5601 {docker-repo}:{version}
    docker run --link 970945a28453:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:7.2.0
```

同样的，Kibana执行上面的命令会输出启动信息，当然也可以加入`-d`参数使Kibnan在后台运行：
```
    docker run -d --link 970945a28453:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:7.2.0
```

### 安装Metricbeat

拉取镜像：
```
    docker pull docker.elastic.co/beats/metricbeat:7.2.0
```

安装、运行Metricbeat：
```
    docker run \
    docker.elastic.co/beats/metricbeat:7.2.0 \
    setup -E setup.kibana.host=kibana:5601 \
    -E output.elasticsearch.hosts=["elasticsearch:9200"]
```

需要替换上面的kibana和elasticsearch主机，可以通过的`docker inspect`查看Elasticsearch、Kibana容器的IP地址：
```
    docker inspect 970945a28453
    docker run docker.elastic.co/beats/metricbeat:7.2.0 setup -E setup.kibana.host=172.17.0.3:5601 -E output.elasticsearch.hosts=["172.17.0.2:9200"]
```