## Docker 版本变化及安装 ##

Docker 从1.13版本后，分为社区版CE 和企业版 EE： 采用时间线的方式作为版本号

社区版分为：

    stable版本： 每个季度更新一次，如17.06 17.09
    edge版本： 每个月份更新一次 如 17.09 17.10

官方方案：

    https://docs.docker.com/engine/installation/linux/docker-ce/centos/

### docker ce版本安装 ###
##### 1）删除老版本的docker或docker-engine  ####
老的docker称为 docker或docker-engine,如果有安装要先卸载
```shell
# yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-selinux \
    docker-engine-selinux \
    docker-engine
```
##### 2) 安装依赖 #####
安装三个依赖：

    yum-utils
    device-mapper-persistent-data
    lvm2
```shell
# yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
```
配置yum仓库
```shell
# yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
命令执行完会添加/etc/yum.repos.d/docker-ce.repo
```repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://download.docker.com/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://download.docker.com/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://download.docker.com/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://download.docker.com/linux/centos/7/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://download.docker.com/linux/centos/7/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://download.docker.com/linux/centos/7/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg

```
默认只开启了stable仓库，如要开启edge仓库
``` shell
# yum-config-manager --enable docker-ce-edge

禁用
# yum-config-manager --disable docker-ce-edge
```

查看docker版本并安装
``` shell
# yum list docker-ce --showduplicates |sort -r

# yum install docker-ce

# yum install docker-ce-17.09.0.ce
```

启动并设置开机启动
```shell
# systemctl start docker
# systemctl enable docker
```

##### rpm 安装docker #####
在  https://download.docker.com/linux/centos/7/x86_64/stable/Packages/ 下载对应版本的docker的rpm包
``` shell
# wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm

# yum install docker-ce-18.03.0.ce-1.el7.centos.x86_64.rpm

# systemctl start docker
```



