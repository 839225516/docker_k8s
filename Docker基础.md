## Docker 基础 ##
### 一 什么是 docker ###
docker 基于 Linux 内核的 cgroup（控制组），namespace（命名空间），以及 AUFS 类的 Union FS（联合文件系统） 等技术，对进程进行封装隔离，属于 操作系统层面的虚拟化技术。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。

传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

Docker 跟传统的虚拟化方式相比具有众多的优势：

    更高效的利用系统资源：由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。
    更快速的启动时间：Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级启动
    一致的运行环境： Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性
    更轻松的迁移：由于 Docker 确保了执行环境的一致性，使得应用的迁移更加容易


Docker采用标准的C/S架构，client和server既可以运行在同一台机器，也可以通过socket或者restful API来进行通信。  

#### 服务端 docker daemon
docker daemon 以后台方式运行在宿主机，作为服务端接受客户端请求（创建、运行、分发容器）。   
docker deamon 默认监听本地的 unix://var/run/docker.sock套接字，只允许本地的root用户访问。  
可以通过 -H 选项来修改监听方式：
> docker -H 0.0.0.0:1234 -d &   

docker 还支持https认证方式来验证访问。  
ubuntu系统中，docker服务的配置文件在 /etc/default/docker
centos系统中，docker服务的配置文件在 /etc/docker/ 

#### 客户端
客户端默认通过 unix:///var/run/docker.socket套接字与服务端通信。   
如果服务端没有监听默认套接字，则通过 -H 指定端口
> docker -H tcp://127.0.0.1:1234 version

#### 命名空间
命名空间实现了容器的资源的隔离，每个容器都有单独的命名空间，包括进程命名空间、网络命名空间、IPC命名空间、挂载命名空间、UTS命名空间、用户命名空间



### 基本概念 ###
#### 镜像 ####
Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

镜像构建时，采用分层存储技术，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。

#### 容器 ####
镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。

镜像使用的是分层存储，容器也是如此。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。

容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。

按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。

数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。

#### 仓库 #### 
集中的存储、分发镜像的服务，Docker Registry 就是这样的服务


### 基本使用 ###

#### 获取镜像 ####
``` shell 
查询镜像：
# docker search ubuntu

拉取镜像
# docker pull ubuntu:16.04

上传镜像到仓库
# docker push ubuntu:16.04

查看本地镜像
# docker images

删除本地镜像
# docker rmi ubuntu:16.04

查看镜像信息
# docker images ubuntu:16.04
# docker inspect ubuntu:16.04

查看镜像构建的完整记录
# docker history --no-trunc ubuntu:16.04 > build.txt

保存镜像到tar文件
# docker save -o ubuntu.tar ubuntu:16.04

从tar文件还原到本地镜像
# docker load -i ubuntu.tar

修改镜像的tag
# docker tag ubuntu:16.04 ubuntu:new_tag
```

#### 启动容器 #### 
```shell
# docker run -it --rm --name ubuntu ubuntu:16.04 bash 
```
参数说明：

    -i: 交互式操作，前面启动
    -d: 后面启动
    -t: 终端
    --rm: 这个参数指定容器退出后随之将其删除，默认不删除
    --name: 指定containName

```shell 
查看运行的容器
# docker ps -l

关闭/重启容器
# docker stop/restart  {containID/containName}

删除容器
# docker rm  {containID/containName}

连接容器
# docker attach {containID/containName}
# docker exec -it {containID/containName} bash

保存容器到镜像
# docker commit {containID/containName}  镜像仓库/镜像名
# docker commit ubuntu Test/ubuntu:test
# docker commit -m='A new imiage' --author='test' ubuntu Test/ubuntu:test

查看容器的日志
# docker logs {containID/containName}

查看容器里的进程信息
# docker top {containID/containName}

从容器里拷贝文件/目录到本地
# docker cp {containID/containName}:/{container_path}  {local_path}
```

### 镜像加速器 ###
国内从 Docker Hub 拉取镜像有时会遇到困难，此时可以配置镜像加速器

Docker 官方加速器： https://registry.docker-cn.com

#### ubuntu,debian,centos7 ####
对于使用systemd的系统，编辑 /etc/docker/daemon.json中写入
```json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```
之后重启docker服务
```shell
# systemctl daemon-reload
# systemctl restart docker
```
