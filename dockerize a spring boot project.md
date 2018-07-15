---
title: docker化spring boot项目
date: 2018-07-15 08:00:41
tags: 实践
---

这里介绍如何docker 化一个spring boot项目，现在我有一个纯净的[spring boot项目](https://github.com/HongkaiWen/war2image/archive/pure-web.zip)。这个项目只有一个功能，进入首页的时候，打印关于项目的配置信息：

```shell
➜  war2image git:(master) ✗ curl http://localhost:8888/
EnvConfig(envName=hello, dbIp=1.1.2.3)
```

配置信息来自于 `application.yaml` ：

```yaml
env:
  db-ip: 1.1.2.3
  env-name: hello
```

通过`IndexController` 来暴露：

```java
@RestController
public class IndexController {
	@Autowired
    private EnvConfig envConfig;

    @GetMapping("/")
    public String index(){
        return envConfig.toString();
    }
}
```

### 容器化的事项

- 容器化目的是后续通过镜像发布，所以项目的配置信息不能打死在镜像里，需要外部化
- 进入镜像的tomcat项目需要做最基本的jvm调优（因为jvm默认设置堆内存为物理内存的1/4,进入容器后，同样会设置为物理机的1/4，但是容器内存大小一般是小于物理机的，所以这个地方必须对xmx做一个限制，否则容器会被cgroup kill掉）
- 要维护好方便的容器化的相关文档和脚本

### 项目目录结构

```
/
	docker/
	Dockerfile
	build.sh
	README.MD
	src/
	pom.xml
```

- Docker 目录存放构建相关的文件
- Dockerfile是项目镜像化的描述文件
- build.sh是镜像化的工具，每次构建镜像通过执行此文件即可

### 添加profile

**配置文件**

application.yaml

```yaml
spring:
  profiles:
    active: @env.flag@
```

Application-local.yaml

```yaml
env:
  env-name: test
  db-ip: 10.2.3.4
```

Application-container.yaml

```yaml
env:
  env-name: {{ .Env.ENV_NAME }}
  db-ip: {{ .Env.DB_IP }}
```

这个地方规划了两个profile，一个是用来本地开发，另外一个是用来容器化部署。

注意 `.Env.ENV_NAME` 是dockerize的语法，表示这个地方的配置来自环境变量 `ENV_NAME`

**pom 配置**

```xml
<profiles>

    <!--****** 本地开发  ****** -->
    <profile>
        <id>local</id>
        <properties>
            <env.flag>local</env.flag>
        </properties>

        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>

    <!--****** 容器部署  ****** -->
    <profile>
        <id>container</id>
        <properties>
            <env.flag>container</env.flag>
        </properties>
    </profile>
</profiles>
```



### 编写Dockerfile

```dockerfile
FROM registry.docker-cn.com/library/tomcat:8.5.27

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 设置字符集
ENV LANG C.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL C.UTF-8

# 删除默认的tomcat里内置的项目
RUN rm -rf /usr/local/tomcat/webapps/*

#安装 dockerize
RUN cd /tmp/ && wget https://github.com/jwilder/dockerize/releases/download/v0.6.1/dockerize-linux-amd64-v0.6.1.tar.gz
RUN tar -C /usr/local/bin -xzvf /tmp/dockerize-linux-amd64-v0.6.1.tar.gz  && rm /tmp/dockerize-linux-amd64-v0.6.1.tar.gz


# 拷贝启动脚本
COPY ./docker/startApp.sh   /app_home/bin/start.sh

# copy war
COPY war2image.war /usr/local/tomcat/webapps/ROOT/
RUN unzip -o  /usr/local/tomcat/webapps/ROOT/war2image.war -d /usr/local/tomcat/webapps/ROOT/
RUN rm -rf /usr/local/tomcat/webapps/dolphin-ops/war2image.war


# 暴露8080端口
EXPOSE 8080

# 启动容器执行的命令
CMD bash /app_home/bin/start.sh
```

**基础镜像**

`registry.docker-cn.com/library/tomcat:8.5.27` 来自国内镜像仓库，比docker hub的快很多。（几乎所有的官方镜像，这个仓库里都有的，只要在官方镜像前面加上前缀`registry.docker-cn.com/library/`即可）

**设置时区**

线上项目，时间这种东西很重要，避免不必要的麻烦。

**设置字符集**

这个也没什么好解释，尤其在我们中文圈，乱码问题太烦。

**安装 dockerize**

这个和外部化配置有关系，虽然spring boot项目原生支持外部化配置到环境变量，但这个地方我选择一个更加通用的[外部化方案](https://github.com/jwilder/dockerize)，他可以基于环境变量对配置文件进行模板化的替换，这种方案与语言无关、与框架无关。

此处仅仅是安装这个工具，具体使用会体现在项目启动过程中；这个工具的包是通过wget下载的，也可以提前把这个包放到项目工程里，不过注意不要让maven的resource插件对这个文件进行过滤，否则二进制文件会被破坏。

**拷贝启动脚本**

项目启动之前需要dockerize介入，把配置文件进行替换，所以启动脚本我需要自定义一下。

### 编写启动脚本

```shell
#!/bin/bash

# 通过dockerize工具动态替换配置文件中的信息
dockerize   -template /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application-container.yaml:/usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application-container.yaml

chmod 774 /usr/local/tomcat/bin/catalina.sh
/usr/local/tomcat/bin/catalina.sh run
```

这个脚本的工作就是通过dockerize工具，基于环境变量，动态替换了application-container.yaml文件里的需要外部化的配置信息。然后启动tomcat。

### 编写构建脚本

我们不能每次都手动构建docker镜像，写一个构建脚本，完成maven打包到docker构建的所有工作：

```shell
#!/bin/bash
echo -e "\e[1;32m step1: package the resouces ......"
echo -e "\e[0m"
mvn clean package -DskipTests -Pcontainer

if [ $? -ne 0 ]; then
  echo -e "\e[1;31m package the resources is unsuccessful."
  echo -e "\e[0m"
  exit;
fi

echo -e "\e[1;32m step2: copy resouce file to root path ......"
echo -e "\e[0m"

cp ./target/*.war ./war2image.war

echo -e "\e[1;32m step3: build docker image ......"
echo -e "\e[0m"

IMAGE_NAME="hongkaiwen/war2image:1.0"

docker build -t $IMAGE_NAME .

if [ $? -ne 0 ]; then
  echo -e "\e[1;31m build docker is unsuccessful."
  echo -e "\e[0m"
  exit;
else
    echo -e "\e[1;32m SUMMARY: build docker is successful, image name: $IMAGE_NAME"
    echo -e "\e[0m"
fi
```



### 使用

上面的所有工作做完，几个镜像化的、工程化的代码结构就开发完了，[完整代码](https://github.com/HongkaiWen/war2image/archive/v1.0.0.zip)

在代码根目录执行 `.build.sh` 命令，即可开始构建，构建成功后执行`docker images`查看刚刚构建的镜像：

```
➜  war2image git:(master) ✗ docker images | grep war2image
hongkaiwen/war2image                     1.0                 ab1d2ae08be8        15 hours ago        592MB
```

然后通过启动命令启动镜像：

```
docker run -d \
           --name war2image \
           --memory="512m" \
           --restart=always \
           -p 8888:8080 \
           -v /Users/hongkai/tmp/war2image:/usr/local/tomcat/logs \
           -e "JAVA_OPTS=-server -Xms300m -Xmx300m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/tomcat/logs/oom.hprof -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/usr/local/tomcat/logs/gcdetail.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M -XX:+PrintTenuringDistribution " \
           -e "ENV_NAME=In-Container" \
           -e "DB_IP=1.5.6.8" \
           hongkaiwen/war2image:1.0
```

查看刚刚启动的容器：

```
➜  war2image git:(master) ✗ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                    NAMES
ff1f3c949158        hongkaiwen/war2image:1.0   "/bin/sh -c 'bash /a…"   15 hours ago        Up About an hour    0.0.0.0:8888->8080/tcp   war2image
```



关于本项目的操作，可以参考代码中的[README.md](https://github.com/HongkaiWen/war2image/blob/master/README.md) 关于docker 基本操作可以参考 [DOCKER 入门](https://hongkaiwen.gitee.io/kuzan/2018/07/12/DOCKER%20%E5%85%A5%E9%97%A8/)



### DOCKER jvm 调优

上面的启动命令里有一句：

```
-e "JAVA_OPTS=-server -Xms300m -Xmx300m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/tomcat/logs/oom.hprof -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/usr/local/tomcat/logs/gcdetail.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M -XX:+PrintTenuringDistribution " \
```

因为这个是一个介绍实践方案的文章，所以这个地方也要提一下。运行时项目，尤其生产环境，jvm有些参数还是要配置一下的：

`-server` server 模式运行项目，会在项目启动时做jit优化，效果是启动慢了一点，运行时快了一点。

`xmx` 在docker里跑jvm这个必须设置，否则jvm会根据宿主机的内存做gc，运行一段时间项目一定会被cgroup kill掉

`gc 日志` 这个生产环境强烈建议配置

`oom 时dump文件` 这个也很有用

上面的具体参数配置我就不一一介绍了，不是本文主题。