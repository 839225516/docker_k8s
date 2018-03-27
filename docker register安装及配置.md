## docker registry

服务器环境：

    CentOS Linux release 7.2.1511 (Core)
    dokcer 17.09.0-ce

### docker registry 安装及配置 ###
#### 1) docker 安装 #### 
``` shell
# yum install docker -y
# systemctl enable docker
# systemctl start docker
```
#### 2）registry安装 ####
```shell 
# docker pull registry:v2.6.2
```
启动容器，并修改配置文件，开启删除tag索引配置

    delete: 
      enabled: true

``` shell 
# docker run -d -p 5000:5000 --name registry registry:v2.6.2

# docker exec -it registry sh
```
vim /etc/docker/registry/config.yml
```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```
保存修改的容器，并删除已启动的registry
```shell
# docker commit registry  registry:v2.6.2_1
# docker stop registry
# docker rm registry
```
启动修改后的容器，将仓库目录/var/lib/registry挂载到本地的/data/docker/registry
```shell
# docker run -d -v /data/docker/registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry:v2.6.2_1
```

配置docker使用私有registry

    --insecure-registry 172.20.8.199:5000

vim /usr/lib/systemd/system/docker.service
```shell
ExecStart=/usr/bin/dockerd --insecure-registry 172.20.8.199:5000 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```

镜像删除操作：
##### 1）查看tag list ######
```shell
# curl -I -X GET http://172.20.8.199:5000/v2/centos/tags/list
```

##### 2) 查看centos:latest的 manifests #####
```shell
# curl --header "Accept: application/vnd.docker.distribution.manifest.v2+json" -I -X GET http://172.20.8.199:5000/v2/centos/manifests/latest
```

##### 3）根据manifests删除镜像的tag索引 ######
```shell
# curl -I -X DELETE   http://172.20.8.199:5000/v2/centos/manifests/sha256:6b44676adc1ba2b09972761e999928280717c7babd760bdd2f4a90595e2c09ea
```

##### 5）删除镜像的物理文件 ######
```shell 
# docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```