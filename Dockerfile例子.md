## Dockerfile 例子 ##

#### Centos 系统初始化 ####
```dockerfile
FROM centos:latest
MAINTAINER TEST

ENV LANG en_US.UTF-8 
RUN rpm --rebuilddb \
    && yum -y install \
        vim \
        tar \
        net-tools \
    && rm -rf /var/cache/yum/* \
    && yum clean all \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```

#### jdk 安装 ####
```dockerfile
#jdk1.8.0_151
FROM 172.20.8.199:5001/library/centos:latest
MAINTAINER test

add jdk-8u151-linux-x64.tar.gz /usr/local/

# 环境变量设置
ENV JAVA_HOME /usr/local/jdk1.8.0_151
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH .:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
ENV PATH $PATH:$JAVA_HOME/bin
```

#### tomcat ####
```dockerfile
FROM 172.20.8.199:5001/library/centos:jdk1.8.0_151
MAINTAINER test

# apache-tomcat-8.5.24 for nocard
RUN mkdir -p /data/
ADD apache-tomcat-8.5.24.tar.gz /data/

# 指定工作目录
WORKDIR /data

CMD ["apache-tomcat-8.5.24/bin/catalina.sh","run"]
```

#### nodejs 安装 ####
```dockerfile
#nodejs:v10.16
FROM 172.20.8.199:5001/library/centos:latest
MAINTAINER test

# code & Timezone
ENV LANG en_US.UTF-8
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime

# install nodejs
ADD node-v10.16.0-linux-x64.tar.gz /usr/local/
ENV NODEJS_HOME /usr/local/node-v10.16.0-linux-x64
ENV PATH $PATH:$NODEJS_HOME/bin
```

