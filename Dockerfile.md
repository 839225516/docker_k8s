Dockerfile定制镜像
=================

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

### dockerfile 结构 ###
```dockerfile
#第一行必须指令基于的基础镜像
From centos:latest

#维护者信息
MAINTAINER test  docker_user@mail.com

# 镜像的操作指令
RUN rpm --rebuilddb \
    && yum -y install \
        vim \
        tar \
        net-tools \
    && rm -rf /var/cache/yum/* \
    && yum clean all
ENV LANG en_US.UTF-8 
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime

#容器启动时执行指令
#CMD [xxx]
```

### dockerfile 的指令 ###
#### FROM 指定基础镜像 ####
```dockerfile
FROM busybox:latest
```

#### RUN 执行命令 ####
```dockerfile
RUN apt-get update
RUN mkdir /data/
```
为了使构建的镜像小，应该合并多条RUN语句到一条，减小构建的层数

#### CMD 指令 #### 
```dockerfile
# 推荐用法
CMD ["executable","param1","param2"]

# 在/bin/sh上执行
CMD command param1 param2

# 提供给ENTRYPOINT做默认参数
CMD ["parma1","param2"]
```
每个容器只能执行一条CMD命令，多个CMD命令时，只最后一条被执行

#### EXPOSE 端口暴露 ####
```dockerfile
EXPOSE 22 80 8443
```
告诉Docker服务端容器暴露的端口号

#### ENV 环境变量 ####
```dockerfile
ENV PATH /usr/local/bin:$PATH
```

#### ADD 复制指定文件到容器中，tar压缩文件会自动解压 ####
```dockerfile
ADD myfile.tar.gz /data/
```

#### COPY 复制指定文件到容器，当使用本地目录为源目录时，推荐使用 COPY ####
``` dockerfile
COPY myfile /data/
```

#### ENTRYPOINT 容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖 ####
```dockerfile
ENTRYPOINT [“executable”, “param1”, “param2”]
ENTRYPOINT command param1 param2 
```

#### VOLUME 创建一个可以从本地主机或其他容器挂载的挂载点 ####
```dockerfile
VOLUME ["/data"]
```

#### USER 指定运行容器时的用户名或UID，后续的 RUN 也会使用指定用户 ####
```dockerfile
USER daemon
```

#### WORKDIR 工作目录 ####
```dockerfile
WORKDIR /data/
```

### Build dockerfile ###
```shell 
# docker build -t 172.20.8.199:5000/jdk:1.8 .
```
一般来说，应该会将 Dockerfile 置于一个空目录下，或者项目根目录下。目录里面不要有多余的文件。因为docker build时会装整个目录打包