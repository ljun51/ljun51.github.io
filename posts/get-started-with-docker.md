---
layout: post
title: Get started with Docker
description: docker快速入门
date: 2019-06-13
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
![app-in-browser][3]

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
        replicas: 5
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

这个`docker-compose.yml文件告诉Docker需要做下面的事：
* 从Docker仓库中拉取镜像
* 运行该镜像的5个实例作为一个服务命名为`web`，限制每个实例最多使用10%的CPU资源、50MB RAM
* 如果失败立即重启容器
* 映射主机的4000端口到`web`的80端口
* `web`服务的容器实例通过叫做`webnet`的负载均衡网络共享80端口
* 使用`webnet`网络的默认设置，webnet是一种负载均衡网络

#### 运行负载均衡应用

在运行`docker stack deploy`命令之前需要执行如下命令，如果没有执行`docker stack init`，会返回"this node is not a swarm manager."。
```shell
    $ docker swarm init
```

给应用取一个名称`getstartedlab`:
```shell
    $ docker stack deploy -c docker-compose.yml getstartedlab
    $ docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
    onevzfj9e50b        getstartedlab_web      replicated          5/5                 ljun51/docker:part2   *:4000->80/tcp
```

也可以通过运行`docker stack services`加服务名查看：
```shell
    $ docker stack services getstartedlab
```

查看任务服务：
```shell
    $ docker services ps getstartedlab_web
```

应用伸缩部署：
```shell
    $ docker stack deploy -c docker-compose.yml getstartedlab
```

关闭应用和swarm：
```shell
    $ docker stack rm getstartedlab
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

### 创建Swarm集群

使用`docker-machine`创建一组VirtualBox驱动的虚拟机：
```shell
    $ docker-machine create --driver virtualbox myvm1
    $ docker-machine create --driver virtualbox myvm2
```

现在创建了`myvm1`、`myvm2`两个虚拟机，查看虚拟机列表和IP地址：
```shell
    $ docker-machine ls

    NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
    myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.7   
    myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.7
```

### 初始化SWARM、添加节点

第一台机器作为管理节点，执行管理和认证命令，让工作机加入swarm，第二台机器作为工作机。可以使用`docker-machine ssh`给虚拟起发送命令，使用`docker swarn init`命令让`myvm1`成为swarm管理节点：
```shell
    $ docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>"
    Swarm initialized: current node <node ID> is now a manager.

    To add a worker to this swarm, run the following command:

    docker swarm join \
    --token <token> \
    <myvm ip>:<port>

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> #### 2377和2376端口
> 运行`docker swarm init`和`docker swarm join`带2377端口（2377是swarm的管理节点），不带端口则使用2377作为默认端口。
>
> 命令`docker-machine ls`返回IP地址和2376端口，2376是Docker守护进程端口。不要让其他程序占有这个端口，否则可能导致其他错误。

> #### 使用SSH遇到问题，尝试使用--native-ssh参数
> Docker Machine带这个参数将使用本机的SSH，如果给swarm管理节点发送命令碰到什么问题可以尝试使用这个命令：
> ```
>    docker-machine --native-ssh ssh myvm1 ...
> ```

运行`docker swarm init`将返回预配置的`docker swarm join`命令，运行这个命令可以将其他节点加入swarm集群，如将`myvm2`通过`docker-machine ssh`加入新的swarm作为工作节点：

```shell
    $ docker-machine ssh myvm2 "docker swarm join \
    --token <token> \
    <ip>:2377"

    This node joined a swarm as a worker.
```

在管理节点运行`docker node ls`查看swarm中的节点：
```shell
    $ docker-machine ssh myvm1 "docker node ls"
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
    rjfae3go6ratysu7dwrjgc5x8 *   myvm1               Ready               Active              Leader              18.09.7
    ibj6aq66bndpl27vfaoru9jaq     myvm2               Ready               Active                                  18.09.7
```

### 在swarm集群部署应用

到目前为止，你都是使用`docker-machine ssh`命令和虚拟起交互；也可以使用`docker-machine env <machine>`和Docker守护进程通信。这种方式对下面的步骤更适合，它允许你使用本地`docker-compose.yml`文件部署远程应用而不必来回拷贝。运行`docker-machine env myvm1`获取命令配置shell和`myvm1`交互：
```shell
    $ docker-machine env myvm1
    export DOCKER_TLS_VERIFY="1"
    export DOCKER_HOST="tcp://192.168.99.100:2376"
    export DOCKER_CERT_PATH="/Users/sam/.docker/machine/machines/myvm1"
    export DOCKER_MACHINE_NAME="myvm1"
    # Run this command to configure your shell:
    # eval $(docker-machine env myvm1)
```

运行给定命令以配置shell以与`myvm1`交互：
```shell
    $ eval $(docker-machine env myvm1)
```

运行`docker-machine ls`验证`myvm1`现在是活动状态，如星号标识所示：
```shell
    $ docker-machine ls               
    NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
    myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.7   
    myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.7
```

运行下面的命令在`myvm1`部署应用：
```shell
    $ docker stack deploy -c docker-compose.yml getstartedlab
```

使用docker命令查看服务在`myvm1`、`myvm2`分发：
```shell
    docker stack ps getstartedlab
    ID                  NAME                         IMAGE                             NODE                DESIRED STATE       CURRENT STATE            ERROR                              PORTS                      
    yxfseer3nsn5        getstartedlab_web.1          ljun51/get-started:part2          myvm2               Running             Running 14 seconds ago                                      
    jq4pignep3em        getstartedlab_web.2          ljun51/get-started:part2          myvm1               Running             Running 14 seconds ago                                      
    v28dyfmqatp9        getstartedlab_web.3          ljun51/get-started:part2          myvm2               Running             Running 14 seconds ago                                      
    bu3fs3iys03t        getstartedlab_web.4          ljun51/get-started:part2          myvm2               Running             Running 14 seconds ago                                      
    fcy2mo24ixam        getstartedlab_web.5          ljun51/get-started:part2          myvm1               Running             Running 14 seconds ago
```

你可以使用`myvm1`或`myvm2`的IP访问集群，创建的网络在集群之间共享并负载平衡。运行`docker-machine ls`获取虚拟机的IP地址，并在浏览器上访问其中任何一个，点击刷新。有5个可能的容器ID都是随机循环的，这表明实现了负载平衡。

![app-in-browser-swarm][5]

### 清理并重启swarm

关闭stack使用`docker stack rm`：
```shell
    $ docker stack rm getstartedlab
```

取消docker-machine shell环境变量设置
```shell
    $ eval $(docker-machine env -v)
```

重启docker machine：
```shell
    $ docker-machine start <machine-name>
```

比如：
```shell
    $ docker-machine start myvm1
    Starting "myvm1"...
    (myvm1) Check network to re-create if needed...
    (myvm1) Waiting for an IP...
    Machine "myvm1" was started.
    Waiting for SSH to be available...
    Detecting the provisioner...
    Started machines may have new IP addresses. You may need to re-run the `docker-machine env` command.
```

这一部分涉及到的命令：
```shell
    docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
    docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
    docker-machine env myvm1                # View basic information about your node
    docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
    docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
    docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
    docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
    docker node ls                # View nodes in swarm (while logged on to manager)
    docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
    docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
    docker-machine ls # list VMs, asterisk shows which VM this shell is talking to
    docker-machine start myvm1            # Start a VM that is currently not running
    docker-machine env myvm1      # show environment variables and command for myvm1
    eval $(docker-machine env myvm1)         # Mac command to connect shell to myvm1
    & "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression   # Windows command to connect shell to myvm1
    docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
    docker-machine scp docker-compose.yml myvm1:~ # Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
    docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app using ssh (you must have first copied the Compose file to myvm1)
    eval $(docker-machine env -u)     # Disconnect shell from VMs, use native docker
    docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
    docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
```

## 第5部分：Stacks

### 加入新服务并重新部署

将服务添加到docker-compose.yml文件很容易。首先，添加一个免费的可视化服务，看看swarm如何调度容器。

docker-compose.yml
```yml
    version: "3"
    services:
    web:
        # replace username/repo:tag with your name and image details
        image: ljun51/getstartedlab:part2
        deploy:
        replicas: 5
        restart_policy:
            condition: on-failure
        resources:
            limits:
            cpus: "0.1"
            memory: 50M
        ports:
        - "80:80"
        networks:
        - webnet
    visualizer:
        image: dockersamples/visualizer:stable
        ports:
        - "8080:8080"
        volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"
        deploy:
        placement:
            constraints: [node.role == manager]
        networks:
        - webnet
    networks:
    webnet:
```

在管理节点重新运行`docker stack deploy`命令，更新服务：
```shell
    $ docker stack deploy -c docker-compose.yml getstartedlab
    $ docker stack ps getstartedlab
```
![get-started-visualizer1][6]

### 持久化数据

再次通过相同的工作流程来添加Redis数据库来存储应用数据。

1. docker-compose.yml
```yml
    version: "3"
    services:
    web:
        # replace username/repo:tag with your name and image details
        image: ljun51/getstartedlab:part2
        deploy:
        replicas: 5
        restart_policy:
            condition: on-failure
        resources:
            limits:
            cpus: "0.1"
            memory: 50M
        ports:
        - "80:80"
        networks:
        - webnet
    visualizer:
        image: dockersamples/visualizer:stable
        ports:
        - "8080:8080"
        volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"
        deploy:
        placement:
            constraints: [node.role == manager]
        networks:
        - webnet
    redis:
        image: redis
        ports:
        - "6379:6379"
        volumes:
        - "/home/docker/data:/data"
        deploy:
        placement:
            constraints: [node.role == manager]
        command: redis-server --appendonly yes
        networks:
        - webnet
    networks:
    webnet:
```

2. 在管理节点创建`./data`目录：
```shell
    $ docker-machine ssh myvm1 "mkdir ./data"
```

3. 确保shell配置与`myvm1`正常交互：
```shell
    $ eval $(docker-machine env myvm1)
```

4. 重新运行`docker stack deploy`：
```shell
    $ docker stack deploy -c docker-compose.yml getstartedlab
```

5. 运行`docker service ls`验证3个服务如预期的运行：
```shell
    $ docker service ls
    ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
    x7uij6xb4foj        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
    n5rvhm52ykq7        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
    mifd433bti1d        getstartedlab_web          replicated          5/5                 gordon/getstarted:latest    *:80->80/tcp
```

6. 检查节点服务，如`http://192.168.99.101`，查看访问者计数器：

![app-in-browser-redis][7]

另外，在任一节点的IP地址上检查端口8080上的可视化器，并注意看到redis服务与web和可视化器服务一起运行。
![visualizer-with-redis][8]

## 第6部分：Deploy your app

*[https://docs.docker.com/get-started][9]*

[1]: https://www.abc12366.com
[2]: https://hub.docker.com/_/hello-world
[3]: https://docs.docker.com/get-started/images/app-in-browser.png
[4]: https://hub.docker.com
[5]: https://docs.docker.com/get-started/images/app-in-browser-swarm.png
[6]: https://docs.docker.com/get-started/images/get-started-visualizer1.png
[7]: https://docs.docker.com/get-started/images/app-in-browser-redis.png
[8]: https://docs.docker.com/get-started/images/visualizer-with-redis.png
[9]: https://docs.docker.com/get-started