---
title: "Java：程序的部署和 Nginx 负载均衡"
author: "Chenghao Zheng"
tags: ["Java"]
categories: ["Study notes"]
date: 2020-04-21T11:30:47+01:00
draft: false
---

本篇介绍 Java 程序部署的三种方式：exec 插件、jar 包、Docker 方式部署，以及使用 Nginx 实现简单的负载均衡。

# Java 程序的部署

### Maven exec 插件部署

1. 导入 Maven 的 exec 插件：[Maven Repository：exec-maven-plugin](https://mvnrepository.com/artifact/org.codehaus.mojo/exec-maven-plugin)，然后在 pom.xml 中进行配置，使之可以调用外部命令执行 Java 程序并自动加载依赖的 classpath 路径：

```html
<plugin>
<groupId>org.codehaus.mojo</groupId>
<artifactId>exec-maven-plugin</artifactId>
<version>1.6.0</version>
<executions>
	<execution>
		<id>run</id>
		<goals>
			<goal>exec</goal>
		</goals>
		<configuration>
			<executable>java</executable>
			<arguments>
				<argument>-classpath</argument>
	<!-- automatically creates the classpath using all project dependencies,
          also adding the project build directory -->
				<classpath/>
				<argument>com.github.NervousOrange.springboot.Application</argument>
			</arguments>
		</configuration>
	</execution>
</executions>
</plugin>
```

2. `mvn exec:exec@[executionID]` 即可完成程序的部署；`-X` 参数可以用于排查部署时出现的问题，`-DskipTests`  可以跳过测试。
3. windosw下（需使用系统 cmd） `netstat -ano | findstr "8080"` 可以查看当前端口被占用的情况，`tasklist |findstr "进程id号"`  查找对应的进程名称，然后 `taskkill /f /t /im "进程id或者进程名称"` 杀掉当前占用端口的进程

部署效果图：

![](/images/exec部署.png)

### Jar 包形式部署

1. 使用 `mvn package -DskipTests` 把项目打包
2. 用命令行参数改变默认的 Spring 端口：`java -jar ./target/Multiplayer-Online-Blog-Platform-0.0.1-SNAPSHOT.jar --server.port=8081`
3. war 包与 jar 包的区别是没有内嵌 Tomcat 容器，无法直接运行，需要将 war 包放置在 Servlet 容器中才能运行。

### 用 Docker 部署

1. 编写 Dockerfile 将之前打的 jar 包放进去，对外暴露 8080 端口。
```dockerfile
FROM java:openjdk-8u111-alpine
RUN mkdir /app
WORKDIR /app       //工作目录
COPY target/Multiplayer-Online-Blog-Platform-0.0.1-SNAPSHOT.jar /app
EXPOSE 8080
CMD ["java","-jar","Multiplayer-Online-Blog-Platform-0.0.1-SNAPSHOT.jar"]
```
2. 构建一个 Docker 镜像：`docker build .` 随后可以用 `docker tag [imageID] [imageName]:tag` 来修改生成镜像的名字和版本

3. 启动容器：`docker run --name blog-launch8082 -p 8082:8080 -v d://application.properties:/app/config/application.properties blog-launch:1.1`

4. 由于 Docker 容器内外环境完全隔离，这就导致容器内的 Java 程序无法正常连接数据库，需要通过 -v 参数将外部的  application.properties（数据库连接的 URL 中的 localhost 改为当前局域网的 IP） 映射到容器内工作目录的 /config 下。 

   参考：[Spring Boot features：Externalized Configuration](https://docs.spring.io/spring-boot/docs/1.2.2.RELEASE/reference/html/boot-features-external-config.html)

6. `docker exec -it [containerID] bash` 可以进到容器内部排查问题

# Nginx 实现负载均衡

### 负载均衡简介

* 负载均衡 Load balancing 是一种水平扩展的分布式解决方案，它可以将高并发的用户流量分摊到多个应用程序服务器上，以缓解单个服务器的流量压力，改善 Web 应用程序的性能，并且可以有效避免单点故障和容灾事故。

* Nginx 文档中提及的三种负载均衡机制：

> round-robin — requests to the application servers are distributed in a round-robin fashion,
>
> least-connected — next request is assigned to the server with the least number of active connections,
>
> ip-hash — a hash-function is used to determine what server should be selected for the next request (based on the client’s IP address).

### 负载均衡配置

* 从 Docker 启动 Nginx：`docker run -d --name blog-nginx --restart=always -v d://nginx/nginx.conf:/etc/nginx/nginx.conf -p 80:80 nginx:1.17.10`

* 在宿主机上配置好 nginx.conf 然后映射到 Docker 容器内部：

```shell
events { }
http {
    upstream myapp1 {
        server 192.168.217.65:8080;   // 本地局域网
        server 192.168.217.65:8081;
        server 192.168.217.65:8082;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

* 出问题后，可以打印日志 `docker logs [containerID]`，也可以进到 Nginx 容器内部进行排查：`docker exec -it ID bash`，测试下网络连通性等。

* 最后的效果图：

![](/images/负载均衡.png)