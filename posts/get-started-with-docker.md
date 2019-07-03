---
layout: post
title: Get started with Docker
description: docker快速入门
image: 
categories: docker
---
[TOC]

# Docker快速入门

通过本课程您可以知道：
1. 安装Docker环境
2. 构建镜像并作为容器运行
3. 扩展应用以运行多个容器
4. 跨集群分发应用
5. 通过添加后端数据库来堆栈服务
6. 部署应用到生产环境

## 第1部分：Orientation

### 准备Docker环境、验证Docker版本

1. Docker分为社区版和企业版，我们学习Docker一般下载社区版，前往[官方网站][1]下载Docker。运行`docker --version`确定Docker是否安装正确：
```shell
    $ docker --version
    Docker version 18.09.2, build 6247962
```

2. 运行`docker info`（或`docker version`）查看已安装Docker的详细信息：
```shell
    $ docker info
    Containers: 38
    Running: 14
    Paused: 0
    Stopped: 24
    Images: 22
    Server Version: 18.09.2
    Storage Driver: overlay2
    ...
```

### 验证Docker安装

1. 通过运行简单的[hello-world][2] Docker镜像来验证Docker是否可以运行：
```shell
    $ docker run hello-world

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
    ...
```

2. 列出机器中已经下载的镜像：
```shell
    $ docker image ls
    REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
    hello-world                                     latest              fce289e99eb9        6 months ago        1.84kB
```

## 第2部分：Containers

### 使用`Dockerfile`定义容器

在本地机器创建一个空的目录(如`docker-practise`)，进入docker-practise目录，创建一个`Dockerfile`文件，拷贝粘贴下面的内容到这个文件，井号开头的行为注释。
```Dockerfile
    # Use an official Python runtime as a parent image
    FROM python:2.7-slim

    # Set the working directory to /app
    WORKDIR /app

    # Copy the current directory contents into the container at /app
    COPY . /app

    # Install any needed packages specified in requirements.txt
    RUN pip install --trusted-host pypi.python.org -r requirements.txt

    # Make port 80 available to the world outside this container
    EXPOSE 80

    # Define environment variable
    ENV NAME World

    # Run app.py when the container launches
    CMD ["python", "app.py"]
```

### Python应用

Python应用需要创建两个文件，`requirements.txt`和`app.py`，把这两个文件和`Dockerfile`放到相同的目录。
requirements.txt
```
    Flask
    Redis
```

app.py
```python
    from flask import Flask
    from redis import Redis, RedisError
    import os
    import socket

    # Connect to Redis
    redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

    app = Flask(__name__)

    @app.route("/")
    def hello():
        try:
            visits = redis.incr("counter")
        except RedisError:
            visits = "<i>cannot connect to Redis, counter disabled</i>"

        html = "<h3>Hello {name}!</h3>" \
            "<b>Hostname:</b> {hostname}<br/>" \
            "<b>Visits:</b> {visits}"
        return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

    if __name__ == "__main__":
        app.run(host='0.0.0.0', port=80)
```

通过`pip install -r requirements.txt`安装Python的Flask、Redis依赖库。这个app打印出环境变量`NAME`和`socket.gethostname()`主机名。如果Redis环境有问题，会输出异常错误信息。

### 构建应用

现在已经准备好了构建的app，再次确认docker-practise目录下包含下面的文件：
```shell
    $ ls
    Dockerfile		app.py			requirements.txt
```

运行build命令，创建Docker镜像，使用`--tag`或`-t`给镜像打标签：
```shell
    $ docker build --tag=friendlyhello .
    $ docker image ls
    REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
    friendlyhello                                   latest              a6cf558f0806        2 days ago          131MB
```

### 运行应用

运行应用的本地机器4000端口到容器的80端口使用`-p`参数：
```shell
    $ docker run -p 4000:80 friendlyhello
```

在浏览器中访问`http://localhost:4000`将显示一个web网页，当然你也可以使用命令行工具`curl`。
![localhost:4000][3]

上面运行应用会在控制台的前台运行，可以使用`CTRL+C`退出控制台；要在在后台运行应用：
```shell
    $ docker run -d -p 4000:80 friendlyhello
```

执行上面的命令后会返回一串很长的容器ID，且容器常驻后台。使用`docker container ls`可以看到容器ID的缩写：
```shell
    $ docker container ls
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
    0920f38be562        friendlyhello       "python app.py"     15 seconds ago      Up 14 seconds       0.0.0.0:4000->80/tcp   zealous_hellman
```

你将注意到`CONTAINER ID`字段和使用`http://localhost:4000`返回的主机名是一致的。使用`docker container stop`加容器ID结束进程：
```shell
    $ docker container stop 0920f38be562
```

### 共享容器

如果没有Docker帐号，登录[hub.docker.com][4]注册，登录Docker公共仓库：
```shell
    $ docker login
```

给镜像打标记或设置版本使用下面的格式：
```shell
    $ docker tag image username/repository:tag 
```
比如：
```shell
    $ docker tag image ljun51/get-started:part2
```
运行[docker image ls][5]查看最近标记的镜像：
```shell
    $ docker image ls

    friendlyhello                                   latest              a6cf558f0806        3 days ago          131MB
    ljun51/get-started                              part2               a6cf558f0806        3 days ago          131MB
```

发布标记的镜像到Docker仓库中：
```shell
    $ docker push ljun51/get-started:part2
```

从远程仓库中拉取、运行镜像：
```shell
    $ docker run -p 4000:80 ljun51/get-started:part2
```
如果本地没有可用镜像的时候，Docker将从远程仓库拉取。

## 第3部分：Services

#### docker-compose.yml

在推送镜像到Docker Hub之后，创建`docker-compose.yml`文件，文件依赖该镜像
```yml
    version: "3"
    services:
    web:
        # replace username/repo:tag with your name and image details
        image: ljun51/docker:latest
        deploy:
        replicas: 3
        resources:
            limits:
            cpus: "0.1"
            memory: 50M
        restart_policy:
            condition: on-failure
        ports:
        - "4000:80"
        networks:
        - webnet
    networks:
    webnet:
```

#### 运行负载均衡应用

在运行`docker stack deploy`命令之前需要执行如下命令，如果没有执行`docker stack init`，会返回"this node is not a swarm manager."。
```shell
    $ docker swarm init
```

给应用取一个名称`springboot`:
```shell
    $ docker stack deploy -c docker-compose.yml springboot
    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
    onevzfj9e50b        springboot_web      replicated          1/1                 ljun51/docker:0.2   *:4000->80/tcp
```

也可以通过运行`docker stack services`加服务名查看：
```shell
    $ docker stack services springboot
```

查看任务服务：
```shell
    $ docker services ps springboot_web
```

应用伸缩部署：
```shell
    $ docker stack deploy -c docker-compose.yml springboot
```

关闭应用和swarm：
```shell
    $ docker stack rm springboot
    $ docker swarm leave --force
```

本文涉及到的命令：
```shell
    docker stack ls                                            # List stacks or apps
    docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
    docker service ls                 # List running services associated with an app
    docker service ps <service>                  # List tasks associated with an app
    docker inspect <task or container>                   # Inspect task or container
    docker container ls -q                                      # List container IDs
    docker stack rm <appname>                             # Tear down an application
    docker swarm leave --force      # Take down a single node swarm from the manager
```

## 第4部分：Swarms

## 第5部分：Stacks

## 第6部分：Deploy your app

[1]: https://www.abc12366.com
[2]: https://hub.docker.com/_/hello-world
[3]: https://docs.docker.com/get-started/images/app-in-browser.png
[4]: https://hub.docker.com