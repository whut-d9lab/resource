# Dokcer入门文档

## 1 docker介绍

> Docker有什么用呢？对于运维来说，Docker提供了一种可移植的标准化部署过程，使得规模化、自动化、异构化的部署成为可能甚至是轻松简单的事情；而对于开发者来说，Docker提供了一种开发环境的管理方法，包括映像、构建、共享等功能, 将来肯定会让普通用户使用起 
>
> 三大概念
> 1 镜像（image） 运行的问题。项目代码需要制作dockerfile文件
> 2 容器（container） 起到隔离的作用 
> 3 仓库（repository） 类似于jar的中央仓库

## 2 安装

### 2.1卸载docker

```
 sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

### 2.2安装docker(centos7)

```
1 sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
  

2  sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo


3  sudo yum install docker-ce docker-ce-cli containerd.io


4  yum list docker-ce --showduplicates | sort -r

5  sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
#启动docker
6  sudo systemctl start docker

#开机自启
7  sudo systemctl enable docker.service
```

## 3 docker指令使用

### 3.1镜像操作

#### 3.1.1下载镜像

```
docker pull NAME[:TAG]
```

NAME是镜像仓库的名称，TAG是镜像的标签（用于表示版本信息）

#### 3.1.2查看镜像

```
docker images
```

#### 3.1.3删除镜像

```
docker rmi image:tag
```

##### 3.1.4制作镜像

基于springboot的jar制作镜像

1编写 Dockerfile 文件，保存名为Dockerfile

```
FROM java:8
VOLUME /tmp
ADD docker-1.0.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

2 springboot项目打包成jar
3 将jar和Dockerfile上传到linux的同一个目录下
4 使用打包语句,注意后面的一个点 

```
docker build -t  docker-1.0.jar  .
```

### 3.2 容器操作

#### 3.2.1 创建容器

------

- **交互型容器：运行在前台，容器中使用exit命令或者调用docker stop、docker** kill命令，容器停止。
- i:打开容器的标准输入。
- t:告诉docker为容器建立一个命令行终端。
- name:指定容器名称，可以不填(随机)，建议根据具体使用功能命名，便于管理。
- centos:告诉我们使用什么镜像来启动容器。
- /bin/bash:告诉docker要在容器里面执行此命令。

------

- **后台型容器：运行在后台，创建后与终端无关，只有调用docker stop、docker**
- kill命令才能使容器停止。
- 推荐：docker run -p 8080:80 --name nginx（容器名称） -d nginx(镜像名称)
- d:使用-d参数，使容器在后台运行。
- c: 通过-c可以调整容器的CPU优先级。默认情况下，所有的容器拥有相同的CPU优先级和CPU调度周期，但你可以通过Docker来通知内核给予某个或某几个容器更多的CPU计算周期。比如，我们使用-c或者–cpu-shares =0启动了C0、C1、C2三个容器，使用-c/–cpu-shares=512启动了C3容器。这时，C0、C1、C2可以100%的使用CPU资源（1024），但C3只能使用50%的CPU资源（512）。如果这个主机的操作系统是时序调度类型的，每个CPU时间片是100微秒，那么C0、C1、C2将完全使用掉这100微秒，而C3只能使用50微秒。
- -c后的命令是循环，从而保持容器的运行。
- docker ps：可以查看正在运行的docker容器。

------

#### 3.2.2查看容器

```
docker ps: 查看当前运行的容器
docker ps -a:查看所有容器，包括停止的。
```

**标题含义**：

>1. CONTAINER ID:容器的唯一表示ID。
>2. IMAGE:创建容器时使用的镜像。
>3. COMMAND:容器最后运行的命令。
>4. CREATED:创建容器的时间。
>5. STATUS:容器状态。
>6. PORTS:对外开放的端口。
>7. NAMES:容器名。可以和容器ID一样唯一标识容器，同一台宿主机上不允许有同名容器存在，否则会冲突。
>
>- docker ps -l :查看最新创建的容器，只列出最后创建的。
>- docker ps -n=2:-n=x选项，会列出最后创建的x个容器。
>
>

#### 3.2.3 容器启动

```
容器名：docker start docker_run
ID：docker start 43e3fef2266c
```

- –restart(自动重启)：默认情况下容器是不重启的，–restart标志会检查容器的退出码来决定容器是否重启容器。

```
docker run --restart=always --name docker_restart -d centos /bin/sh -c "while true;do echo hello world; sleep;done":
```

- --restart=always:不管容器的返回码是什么，都会重启容器。
- --restart=on-failure:5:当容器的返回值是非0时才会重启容器。5是可选的重启次数。

#### 3.2.4容器停止

```
docker stop [NAME]/[CONTAINER ID]:将容器退出。<br>
docker kill [NAME]/[CONTAINER ID]:强制停止一个容器。
```

#### 3.2.5 容器删除

- 不能够删除一个正在运行的容器，会报错。需要先停止容器。

```
docker rm [NAME]/[CONTAINERID]
```

#### 3.2.6 查看日志

```
docker logs -f Nmae/id
```

## 4 docker修改容器的配置

### 1 进入容器

```
docker exec -it nginx bash
```

### 2 vi或者vim命令（新增）

```shell
apt-get  update
apt-get install vim
```

### 3 文件复制

```
容器内复制到宿主机（主机）
docker cp mycontainer:/opt/testnew/file.txt /opt/test/
主机复制到容器中
docker cp /opt/test/file.txt mycontainer:/opt/testnew/
```

### 4 文件挂载（替换）

一般采用该方式进行替换容器的配置问题文件（推荐）

```shell
nginx配置文件的挂载方式

docker run \
  --name nginx \
  -d -p 80:80 \
  -v /usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /usr/local/docker/nginx/conf/conf.d:/etc/nginx/conf.d \
  nginx
```