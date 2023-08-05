# Docker是什么

Docker是一个开源平台，通过将应用程序隔离到轻量级、可移植的容器中，自动化应用程序的部署、扩展和管理。容器是独立的可执行单元，封装了运行应用程序所需的所有必要依赖项、库和配置文件，可以在各种环境中稳定地运行。

![img.png](img.png)

# 相关名词概念

镜像(image)：Docker镜像好比是一个模板，可以通过这个模板来创建容器服务，tomcat镜像===>run==>tomcat01容器（提供服务器）。通过这个镜像可以创建多个容器（服务运行或项目运行是在容器中）

容器(container)：Docker利用容器技术，独立运行一个或一个组应用，通过镜像来创建。 可以把这个容器理解为一个简易的linux系统

仓库(repository)：存放镜像的地方。仓库分为公有仓库和私有仓库。

Docker_Hub(默认是国外的)，要利用国内容器服务器配置镜像加速。

# 安装Docker

# Docker原理

![img_1.png](img_1.png)

**Docker是怎么工作的？**

Docker是一个Client-Server结构的系统，Docker的守护进程运行在主机上。通过Socket从客户端访问。

Docker/Server接收到Docker-Client的指令，就会执行这个命令。

**Docker为什么比虚拟机快？**

1. Docker有比虚拟机更少的抽象层。

![img_2.png](img_2.png)

2. Docker利用的是宿主机的内核，虚拟机需要是Guest OS。所以，新建一个容器的时候，Docker不需要像虚拟机一样重新加载一个操作系统内核，避免引导。虚拟机是加载Guest OS，分钟级别的，而Docker是利用宿主机的操作系统，省略了这个复杂的过程，秒级别。

# 基本命令

## 帮助命令

```shell
docker version # 查看版本
docker info # 查看docker系统信息，包括镜像和容器数量
docker 命令 --help #帮助命令
```
帮助文档地址：https://docs.docker.com/engine/reference/run/

## 镜像基本命令

```shell
docker images #查看所有本地主机上的镜像
```
![img_3.png](img_3.png)
```shell
#解释
REPOSITORY 镜像的仓库源
TAG        镜像的标签
IMAGE ID   镜像的id
CREATED    镜像的创建时间
SIZE       镜像的大小
# 可选项
-a, --all   列出所有的镜像
-q, --quiet 只显示镜像的id
```
```shell
docker search [过滤项] 镜像名 # 搜索镜像
```
![img_4.png](img_4.png)
```shell
# 可选项
-f, --filter 列出收藏数不小于指定值的镜像
```
```shell
docker pull 镜像名[:tag]  #下载镜像
$ docker pull ubuntu:14.04          # 如果不写tag默认是latest

14.04: Pulling from library/ubuntu
5a132a7e7af1: Pull complete         # 分层下载, docker image的核心, 联合文件系统
fd2731e4c50c: Pull complete
28a2f68d1120: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2  # 签名
Status: Downloaded newer image for ubuntu:14.04
docker.io/library/ubuntu:18.04      # 真实地址                            
```
```shell
docker rmi -f 镜像id/镜像仓库源  # 删除镜像
docker rmi -f 镜像id 镜像id  # 删除多个镜像
docker rmi -f $(docker images -aq) # 删除所有镜像
```
![img_6.png](img_6.png)

## 容器基本命令

```shell

```