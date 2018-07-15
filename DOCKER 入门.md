---
title: Docker 入门
date: 2018-07-12 19:00:00
tags: 总结整理
---

[CAAS](https://blog.docker.com/2016/02/containers-as-a-service-caas/)平台底层依赖docker和k8s，docker在CAAS中解决了什么问题？抛开CAAS, docker设计理念是什么样的？本文目的给大家从纯技术的角度，简单介绍一下docker的方方面面，方便大家入门。后续会抽时间整理docker实践过程中积累下来的每一个知识点的我自己的一个最佳实践。

### docker 解决的问题

**docker**

在我看来，狭义的 [docker](https://www.docker.com/) （不包括docker生态圈），只能能解决 **单机** 层面部署运维复杂度的问题：

- 部署环境一致性
- 部署过程自动化
- 进程隔离
- 资源隔离
- 资源管控
- 简单运维（自动重启...）

其他一些功能，比如跨主机调度docker 容器、容器跨主机通信、监控、负载均衡、服务发现等高端功能，要依赖于docker 生态圈里的其他弟兄，比如cadvisor做监控、swarm做集群调度、跨主机网络插件做网络、nginx做负载。服务发现？可能要考业务层自己实现，比如spring cloud或dubbo等。

**关于k8s**

上文从跨主机调度角度，介绍了docker自家的集群调度方案 swarm，其实这个swarm是比较冷门的，解决集群调度的 `网红` 是k8s。k8s是谷歌家的产品，出了跨主机调度的能力，还有其他很多很多功能，k8s可以理解为docker生态圈里的spring，是一个`全家桶` 工具箱。

- 容器集群管理
- 简单监控
- 自动化运维
- 服务发现
- 服务暴露
- 多场景容器管理
- 跨主机网络支持
- ….. (恕我见识短，了解的也有限)



### 实现原理

回到本文的主题 docker。docker是怎样实现上述的功能的呢？ 其实docker只是对操作系统已有功能做了一个友好的封装：

- 基于操作系统的namespace，做了进程隔离
- 基于cgroup做了资源限制

推荐看一篇[文章](http://dockone.io/article/2941)

上面说的蛮高深的吧？其实大概知道一下就可以了，技术方向有很多，有业务驱动需要把某一个方向做精了再去了解吧。

### 核心概念

- image
- container
- registry

可以做一下类比，image好比程序包，container好比运行时进程，registry好比app store。

源代码可以被构建（docker的构建）变成image，image可以被运行变成container。image也可以被push，存储到镜像仓库，方便集群使用。

[阿里仓库](https://dev.aliyun.com/search.html) [官方仓库](https://hub.docker.com/)

### 操作

**镜像制作**

[镜像制作](https://docs.docker.com/engine/reference/commandline/build/#build-with-url)，一般是基于几个操作系统类的基础镜像来做，比如 [centos](https://hub.docker.com/_/centos/)；也有一些现成的应用容器类的基础镜像，比如：[tomcat](https://hub.docker.com/_/tomcat/)；也有一些成型的开箱即用类的镜像，比如： [mysql](https://hub.docker.com/_/mysql/)、[redis](https://hub.docker.com/_/redis/)

基于基础镜像如何做新的镜像呢？

**黑盒制作**

```
1. 找到目标镜像 docker images
2. 启动并进入容器 docker run -it <image id> /bin/bash
3. 修改配置
4. 退出容器 exit
5. 找到刚才的容器 docker ps -a
6. 生成镜像 docker commit -m "<comment>" <container id><image name>:<version>
7. 找到新生成的镜像 docker images 
8. 启动并进入容器 docker run -it <image id> /bin/bash 检查配置是否生效
9. 删除修改过程中和验证过程中生成的容器
10. 导出镜像文件 docker save -o <file name>.tar <image name>
```

**白盒制作** （基于docker file制作）

黑盒、白盒，我自己起的名字，比较形象，一般推荐基于docker file制作，因为所有的过程都是可追溯的，如果是黑盒的镜像，除非是官方的，一般我也不敢用。

docker file的语法，参考[官网](https://docs.docker.com/engine/reference/builder/#usage)；这里举一个简单的基于官方tomcat镜像做helloworld项目的容器化的例子：

```dockerfile
FROM tomcat:8.5-alphine

# set time zone
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 设置编码方式
ENV LANG C.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL C.UTF-8

# 删除默认的tomcat项目
RUN rm -rf /usr/local/tomcat/webapps/*

#安装 dockerize
COPY ./dockerize-linux-amd64-v0.6.1.tar.gz /tmp/
RUN tar -C /usr/local/bin -xzvf /tmp/dockerize-linux-amd64-v0.6.1.tar.gz  && rm /tmp/dockerize-linux-amd64-v0.6.1.tar.gz

# 自定义启动脚本
COPY ./startup/startApp.sh   /app_home/bin/start.sh

# copy war
COPY helloworld.war /usr/local/tomcat/webapps/ROOT/
RUN unzip -o  /usr/local/tomcat/webapps/ROOT/helloworld.war -d /usr/local/tomcat/webapps/ROOT/
RUN rm -rf /usr/local/tomcat/webapps/ROOT/helloworld.war


# 暴露8080端口
EXPOSE 8080

# 启动容器执行的命令
CMD bash /app_home/bin/start.sh
```

上面的dockerfile把一个简单的helloworld项目构建成了docker镜像，方面后续做容器化部署。

里面用到了[dockerize](https://github.com/jwilder/dockerize)是一个容器化过程中外部化配置文件的方案，后面我会单独写一篇文章，介绍一下war包构建镜像的实践。

构建命令可以通过 `docker build --help` 来了解详细。举个例子：

```Shell
docker build -t $IMAGE_NAME .
```

**容器**

查看容器列表

```shell
docker ps
docker ps -a
```



创建并启动容器

查看帮助 `docker run --help`

样例：

```shell
docker run -d \
       --name helloworld \
       --memory="1024m" \
       --restart=always \
       -p 9091:8080 \
       -v /data:/data/helloworld \
       -e "MYSQL_IP=10.65.3.5" \
       helloworld:1.0.1

```

- `--name` 容器名
- `-d` 表示deamon方式运行，命令执行完即退出
- `--memory` 表示容器最大可用内存
- `--restart` 表示重启策略，上述配置表示挂掉即重启
- `-p` 表示端口映射，内部的8080端口暴露到书主机的9091端口
- `-v` 表示目录挂载，宿主机的目录映射到容器里（默认容器里的目录都是非持久化的，容器销毁，文件消失，如果要持久化保存，简单的做法是把目录挂载出来，可以挂到宿主机，也可以挂到分布式存储）
- `-e` 环境变量

启动/停止容器

```shell
docker start <容器id或容器名>
docker stop <容器id 或容器名>
```



查看容器日志

```shell
docker logs -f --tail=500 <容器id或容器名>
```

- `-f` 标识跟踪查看，实时输出
- `--tail` 表示从日志尾部网上多少行



查看容器资源状态

```shell
docker stats 
docker stats <容器id或容器名>
```



进入容器

```shell
docker run -it <容器id或容器名> /bin/bash
```



拷贝容器文件

```shell
# 从容器内拷贝到宿主机
docker cp <容器id或容器名>:/xxxxx /xxxx
# 从宿主机拷贝到容器内
docker cp /xxxx <容器id或容器名>:/xxxxx
```



删除容器

```shell
docker rm
```



**仓库**

```shell
docker push <镜像名>
docker pull <镜像名>
```



### 结束

现在看来，docker有两个使用场景，一个是分布式集群系统里使用。一个是pc这边作为一些工具或甚至是在家自己快速搭建一些私有nas之类的用途，只要有了docker，真得很方便。

利用docker快速搭建一个mysql作为自己的开发使用如何搞？

第一步，去[APP store](https://dev.aliyun.com/detail.html?spm=5176.1972343.2.2.79825aaaRKy2Xc&repoId=1239) 下一个 `docker pull mysql`

第二步，启动。`docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag`



自己本地化搭建私有的prosesson：

```shell
docker run -it  --name="draw" -p 8080:8080 -p 8443:8443 registry.docker-cn.com/fjudith/draw.io
```



















