---
layout: post
title: Spring Boot with Docker
description: 通过docker构建Spring Boot镜像
image: 
categories: other
---
[toc]

### Spring Boot with Docker

可以通过[Spring Initializr][1]生成Spring Boot骨架，参考[Spring Boot with Docker][2]教程为Spring Boot应用构建Docker镜像。

#### 设置Spring Boot应用

```java
    package io.github.ljun51.demo;

    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;

    @SpringBootApplication
    @RestController
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
        
        @GetMapping("/")
        public String hello() {
            return "hello docker world";
        }
    }

```

#### 手工编译镜像

在项目根目录创建Dockerfile文件

```Dockerfile
    FROM openjdk:8-jdk-alpine
    VOLUME /tmp
    COPY target/*.jar app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]	
```

编译镜像
```shell
    $ mvn package
    $ docker build -t ljun51/docker .

    $ docker images

    $ docker run -ti --entrypoint /bin/sh ljun51/docker
    # ls
```

启动镜像测试
```shell
    $ docker run -p 8080:8080 ljun51/docker

    $ curl http://localhost:8080
    hello docker world
```

#### Dockerfile放在src/main/docker/Dockerfile下

```Dockerfile
    FROM openjdk:8-jdk-alpine
    VOLUME /tmp
    COPY ./target/*.jar app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

```shell
    $ docker rmi ljun51/docker -f
    $ docker build -t ljun51/docker -f src/main/docker/Dockerfile .
    $ docker run -p 8080:8080 ljun51/docker
```

```shell
    $ curl http://localhost:8080
    hello docker world
```

#### 通过参数指定Spring Boot文件

```Dockerfile
    FROM openjdk:8-jdk-alpine
    VOLUME /tmp
    ARG JAR_FILE
    COPY ${JAR_FILE} app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

```shell
    $ mvn package
    $ docker rmi ljun51/docker -f
    $ docker build --build-arg JAR_FILE=target/*.jar -t ljun51/docker:0.1 -f src/main/docker/Dockerfile .
    $ docker run -p 8080:8080 ljun51/docker
```

#### SPRING_PROFILES_ACTIVE指定配置文件

```shell
    $ docker run -e "SPRING_PROFILES_ACTIVE=prod" -p 8080:8080 ljun51/docker
```

#### 推送镜像到仓库

```shell
    $ docker push ljun51/docker
    The push refers to repository [docker.io/ljun51/docker]
    0dc66457dc0d: Pushed
    ceaf9e1ebef5: Mounted from library/openjdk
    9b9b7f3d56a0: Mounted from library/openjdk 
    f1b5933fe4b5: Mounted from library/openjdk
    latest: digest: sha256:a4aa035cee0ddfd4d111c2b12ee9fa8f214d5e302b4416f9e81b4b59a16f2338 size: 1159
```

[1]: https://start.spring.io
[2]: https://spring.io/guides/gs/spring-boot-docker