# docker 之FastDFS安装

## 1.查找Docker Hub上的redis镜像

````
docker search fastdfs
````

![img](https://img-blog.csdnimg.cn/img_convert/aa7fdae08bb9e6ce49fbc47ed9074df3.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 2.拉取镜像

```
docker pull delron/fastdfs #拉取最新版本
```



![img](https://img-blog.csdnimg.cn/img_convert/47867d0f9bab5c7061cdc0f646d8a9c5.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

##  3.查看镜像

```
docker images
```

![img](https://img-blog.csdnimg.cn/img_convert/40c0416b0203a86201fe80caa9e88e36.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 4.开放端口

````shell
重启防火墙

systemctl restart firewalld



firewall-cmd --zone=public --permanent --add-port=8888/tcp

firewall-cmd --zone=public --permanent --add-port=22122/tcp

firewall-cmd --zone=public --permanent --add-port=23000/tcp



再次重启防火墙

systemctl restart firewalld
````



## 5.使用docker镜像构建tracker容器（跟踪服务器，起到调度的作用）：

```shell
docker run -dti --network=host --name tracker -v /var/fdfs/tracker:/var/fdfs -v /etc/localtime:/etc/localtime delron/fastdfs tracker
```

## 6.使用docker镜像构建storage容器（存储服务器，提供容量和备份服务）：

```shell
docker run -dti --network=host --name storage -e TRACKER_SERVER=192.168.56.1:22122 -v /var/fdfs/storage:/var/fdfs -v /etc/localtime:/etc/localtime  delron/fastdfs storage
```

TRACKER_SERVER=本机的ip地址:22122 本机ip地址不要使用127.0.0.1

进入容器

```
docker exec -it storage /bin/bash
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

进入storage容器，到storage的配置文件中配置http访问的端口，配置文件在/etc/fdfs目录下的storage.conf。

![img](https://img-blog.csdnimg.cn/img_convert/aaaf743bb6e4fc154f6852deda264273.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 默认端口是8888，也可以不进行更改。

如果重启后无法启动的会，可能是报下面错误了，手动创建 vi /var/fdfs/logs/storaged.log 文件即可

tail: cannot open '/var/fdfs/logs/storaged.log' for reading: No such file or directory

## 7.配置nginx

进入storage,配置nginx，在/usr/local/nginx目录下，修改nginx.conf文件,默认配置不修改也可以

![img](https://img-blog.csdnimg.cn/img_convert/1a319185df285f0a99e228233ee9dcdd.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)





## 8.测试上传文件

使用web模块进行文件的上传，将文件上传至FastDFS文件系统

 将一张照片（test.png）放置在/var/fdfs/storage目录下，进入storage容器，进入/var/fdfs目录，运行下面命令：

![img](https://img-blog.csdnimg.cn/img_convert/3cb6739aa621d4395d2596f64067087e.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

##  9.开启启动容器

```shell
docker update --restart=always tracker

docker update --restart=always storage
```

## 10.常见问题

storage 无法启动
运行 docker container start storage 无法启动，进行如下操作即可：
可以删除/var/fdfs/storage/data目录下的fdfs_storaged.pid 文件，然后重新运行storage。

## 11 文件上传工具类

 githhub工具类地址： https://github.com/shu616048151/fastdfs.git

![img](https://img-blog.csdnimg.cn/20201007135945390.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NodTYxNjA0ODE1MQ==,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)