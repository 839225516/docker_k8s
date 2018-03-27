## Harbor 安装及配置 ##
Harbor是VMWare开源的一个docker registry

githua地址： https://github.com/vmware/harbor/releases

#### harbar安装 ####
离线安装最新版本harbor v1.4.0
```shell
# wget https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz
# tar zxf harbor-offline-installer-v1.4.0.tgz
# cd harbor
```

修改配置文件 harbor.cfg

    #设置域名或ip,默认端口是443，改成80(http)
    hostname = 172.20.8.15:80

    # 设置访问协议 http/https
    ui_url_protocol = http

安装docker-compose
```shell
# yum install pip-python
# pip install docker-compose
```

安装harbor：运行install.sh 脚本，导入镜像文件到本地，并启动容器级
```shell
# ./install.sh
```
Harbor的所有容器是通过docker-compose来管理，定义在配置文件docker-compose.yml
``` shell 
# cd harbor
# docker-compose up -d 

停止
# docker-compose down -v
```
#### 验证: ####
1) 访问 http://172.20.20.15:80  默认admin/Harbor12345

2) docker client验证
修改dcoker启动脚本：
```shell
# vi /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd  --insecure-registry 172.20.20.15:80
```
启动docekr 
```shell
# systemctl daemon-reload
# service docker restart
```
登录harbor并上传镜像：
```shell 
# docker  login 172.20.20.15:80
>> admin
>> {passwd}

退出登录
# docker logout 172.20.20.15:80

上传镜像
     前提条件：用户有项目的权限
构建镜像再上传
# docker build -t 172.20.20.15:80/library/centos:latest .
# docker push 172.20.20.15:80/library/centos:latest

或拉取远程镜像，打tag再上传
#  docker pull  centos:latest
# docker tag centsot:latest 172.20.20.15:80/library/centos:latest
#  docker push 172.20.20.15:80/library/centos:latest
```
#### 删除harbor的镜像 ####
1) 网页端先删除镜像tag
2) 删除物理文件
```shell 
# docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect --dry-run /etc/registry/config.yml

# docker run -it --name gc --rm--volumes-from registry vmware/registry:2.6.2-photon garbage-collect  /etc/registry/config.yml
```
-----------------
#### Harbor 切换成https 访问  ####
1) 准备证书 secret.crt , secret.key
2) 修改配置文件 harbor.cfg
```shell 
#set ui_url_protocol
ui_url_protocol = https

#The path of cert and key files for nginx, they are applied only the protocol is set to https 
ssl_cert = /root/cert/secret.crt
ssl_cert_key = /root/cert/secret.key

```
3) 重启harbor
``` shell
# ./prepare
# docker-compose down -v 
# docker-compose up -d 
```





