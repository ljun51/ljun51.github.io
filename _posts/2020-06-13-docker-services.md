---
layout: post
title: Docker Services
description: 通过docker构建Spring Boot镜像
image: 
categories: other
---
[toc]

### Docker Services

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