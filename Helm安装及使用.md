## Helm安装及使用 ##

Helm是一个类似于yum/apt/homebrew的Kubernetes应用管理工具。Helm使用Chart来管理Kubernetes manifest文件

Helm的三个基本概念：

    Chart：Helm应用（package），包括该应用的所有Kubernetes manifest模版，类似于YUM RPM或Apt dpkg文件
    Repository：Helm package存储仓库
    Release：chart的部署实例，每个chart可以部署一个或多个release

### Helm安装 ###
依赖:

    Socat
	kubectl
```shell
# yum install -y socat 
k8s node 全局节点
```

#### 1）安装Helm客户端： ####
```shell
# wget https://kubernetes-helm.storage.googleapis.com/helm-v2.7.2-linux-amd64.tar.gz
# tar zxf helm-v2.7.2-linux-amd64.tar.gz
# cp linux-amd64/helm /usr/local/bin/helm
```
#### 2） RBAC ####
tiller部署在Kubernetes 1.8上，Kubernetes APIServer开启了RBAC访问控制，所以我们需要创建tiller使用的service account: tiller并分配合适的角色给它

创建rbac-config.yaml文件：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
``` shell
# kubectl create -f rbac-config.yaml
```
#### 3）服务端tiller安装 ####
安装Tiller的最简单方式是helm init, 该命令会检查helm本地环境设置是否正确，helm init会连接kubectl默认连接的kubernetes集群（可以通过kubectl config view查看），一旦连接集群成功，tiller会被安装到kube-system namespace中，会在当前目录下创建helm文件夹即~/.helm，在k8s集群kube-system namespace下安装了deployment tiller-deploy和service tiller-deploy
```shell 
# helm init --service-account tiller --upgrade

# helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.7.2 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

#### 4） 检查安装是否成功 ####
```shell 
# helm version
```

### Helm 的基本操作 ###

#### 1) 仓库 ####
查看仓库
``` shell
# helm repo list
# helm repo update
```
添加本地仓库
```shell
# helm repo add monocular https://kubernetes-helm.github.io/monocular
```
#### 2） chart ####
``` shell
查询charts
# helm search

查看指定chart的基本信息
# helm inspect monocular/monocular

# helm install monocular/monocular

安装打包的chart
# helm install ./nginx-1.2.3.tgz

安装打包目录
# helm install ./nginx

安装chart包URL
# helm install https://example.com/charts/nginx-1.2.3.tgz

创建chart
# helm create my-chart

--dry-run --debug 打印chart文件,不执行安装
# helm install . --dry-run --debug

install 时用 -n指定 release的名字
# helm install . -n manageweb
```

管理release
```shell 
查看已部署的release
# helm list

查看已删除的release
# helm ls --deleted

查看指定release的状态
# helm status manageweb

查看指定release的配置文件值
# helm get values manageweb


查看历史版本的序列
# helm hist manageweb

查看指定release的历史版本部署时部分配置信息
# helm get --revision 1 manageweb 

回滚release到指定发布序列
# helm rollback [release] [revision]
# helm rollback --debug manageweb 1

升级某个release
# helm upgrade posp-proxy-verify . --recreate-pods

删除release
# helm delete manageweb

完全删除release, --purge
# helm del --purge posp-accp-finance

```

------------------------------------------
#### heml 私有仓库搭建  ####
创建本地chart仓库
``` shell 
# helm serve –address 0.0.0.0:8879 –repo-path /data/helm-repo &
```

打包一个charts,accp-finance-0.1.5.tgz
``` shell 
# helm package [flags] [CHART_PATH] [...]
# helm package accp-finance
```

上传chart到charts仓库，并将chart的metadata记录在index.yaml文件
```shell
# mv accp-finance-0.1.5.tgz /data/helm-repo
# helm repo index accp-finance --url http://172.20.2.236:8879
```

查看charts 
```shell
# helm repo update
# helm search accp-finance
```


