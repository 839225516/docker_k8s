##### kubeadm安装k8s
> https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/

环境：

    kubernetes v1.13
    kube-master     172.20.20.247      Centos7
    kube-node1      172.20.20.248      Centos7
    kube-node2      172.20.20.249      Centos7

1. 修改主机名
```shell
hostnamectl set-hostname kube-master
hostnamectl set-hostname kube-node1
hostnamectl set-hostname kube-node2

# 添加hosts
cat <<EOF >> /etc/hosts
172.20.20.247        kube-master
172.20.20.248        kube-node1
172.20.20.249        kube-node2
EOF
```

2. 关闭 firewalld, selinux, swap 
```shell
systemctl stop firewalld && systemctl disable firewalld

setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux

swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
```

3. 修改系统net参数
```shell
# 创建/etc/sysctl.d/k8s.conf文件，添加如下内容
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 执行命令使修改生效
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge
```

4. kube-proxy 开启ipvs的前置条件
由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

    ip_vs
    ip_vs_rr
    ip_vs_wrr
    ip_vs_sh
    nf_conntrack_ipv4

在所有的kubernetes节点上执行脚本：
```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules 
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

5. ssh免密登录
```shell
cd ~ && mkdir .ssh && chmod 700 .ssh && cd ~/.ssh && ssh-keygen -t rsa -f ./id_rsa  -P ""

# cp 公钥到 172.20.20.248 172.20.20.249
ssh-copy-id -i ~/.ssh/id_rsa.pub 172.20.20.248
ssh-copy-id -i ~/.ssh/id_rsa.pub 172.20.20.249
```

6. 确保各个节点上已经安装了ipset软件包 yum install ipset。    
为了便于查看ipvs的代理规则，最好安装一下管理工具ipvsadm, yum install ipvsadm。    
如果以上前提条件如果不满足，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式
```shell
yum install ipset -y
yum install ipvsadm -y
```

7. 安装docker
Kubernetes从1.6开始使用CRI(Container Runtime Interface)容器运行时接口。默认的容器运行时仍然是Docker，使用的是kubelet中内置dockershim CRI实现

安装docker的yum源
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo


# 查看最新的docker版本
yum list docker-ce.x86_64  --showduplicates |sort -r


yum makecache fast

# 安装最新版 docker-ce
yum install -y docker-ce
```
确认一下iptables filter表中FORWARD链的默认策略(pllicy)为ACCEPT   
Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FORWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信。  

当启动 Docker 服务时候，默认会添加一条转发策略到 iptables 的 FORWARD 链上。策略为通过
（ ACCEPT ）还是禁止（ DROP ）取决于配置 --icc=true （缺省值）还是 --icc=false 。
默认情况下，不同容器之间是允许网络互通的。如果为了安全考虑，可以在 /etc/default/docker 文件中配置 DOCKER_OPTS=--icc=false 来禁止它。   
但k8s集群环境中，必须要打开 --icc=true 

```shell
# 查看 iptables 规则 
iptables -nvL

# 配置 docker 参数
# insecure-registries ： 非安全的私有镜像仓库
# registry-mirrors ：    镜像加速器，加速镜像下载（貌似没什么用，可以不配置）

cat > /etc/docker/daemon.json <<EOF
{
    "icc": true,
    "insecure-registries": ["registry.jlpay.io"],
    "registry-mirrors": ["https://registry.docker-cn.com", "https://docker.mirrors.ustc.edu.cn"]
}
EOF

iptables -P FORWARD ACCEPT

systemctl start docker
systemctl enable docker

iptables -nvL
```

8. 使用 kubeadm 安装 k8s 集群
```shell 
# 安装 kubernetes 的阿里云yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# master 都要装 kubelet kubeadm kubectl; node 节点只用装kebelet 和 kubeadm
yum install -y kubelet kubeadm kubectl 
systemctl enable kubelet
```

###### 生成 kubeadm 的配置文件并初始化安装第一个master节点
注意 VIP=api.k8s.jlpay.com  dns是否能解析，不能则加hosts

kubeadm_install_master.sh
```shell
#!/bin/bash


CP0_IP=172.20.20.247
CP1_IP=172.20.20.248
CP2_IP=172.20.20.249
VIP=api.k8s.jlpay.com
CIDR=10.172.0.0/16


#################### kubeadm 配置文件 ####################################################
#  kubernetesVersion: v1.14.0  指定安装的版本
#  imageRepository: registry.aliyuncs.com/google_containers   指定使用阿里云镜像仓库
#  apiServerCertSANs 填所有的： masterip、lbip 、其它可能需要通过它访问 apiserver 的地址、域名或主机
#  controlPlaneEndpoint: "${VIP}:6443"   这里使用vip作api的https地址
#  podSubnet: "10.172.0.0/16"    指定pod网段，这里注意，后面安装 CNI插件的configmap也要改成这个网段
#  serviceSubnet: 10.96.0.0/12   指定 service 的网段
#  mode: iptables/ipvs           指定 k8s的proxy模式，1.11后默认进行ipvs转发

echo """
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "${VIP}:6443"
apiServer:
  certSANs:
  - ${CP0_IP}
  - ${CP1_IP}
  - ${CP2_IP}
  - ${VIP}
networking:
  # This CIDR is a Calico default. Substitute or remove for your CNI provider.
  podSubnet: ${CIDR}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
""" > /etc/kubernetes/kubeadm-config.yaml

# 根据配置文件安装master节点
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml
```

查看k8s的集群状态
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 查看k8s集群状态
kubectl cluster-info
kubectl get cs
kubectl get po --all-namespaces
kubectl get nodes
```

###### 安装另外两台 mastser
同步 ca 文件到 master:  sync_ca_install_master.sh
```shell
#!/bin/sh

vhost="kube-node1 kube-node2"
usr=root

who=`whoami`
if [[ "$who" != "$usr" ]];then
  echo "请使用 root 用户执行"
  exit 1
fi

echo $who

# 需要从 m01 拷贝的 ca 文件
caFiles=(
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/sa.key
/etc/kubernetes/pki/sa.pub
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/admin.conf
)

pkiDir=/etc/kubernetes/pki/etcd
JOIN_CMD=`kubeadm token create --print-join-command`
for h in $vhost 
do

  ssh ${usr}@$h "mkdir -p $pkiDir"
  
  echo "Dirs for ca scp created, start to scp..."

  # scp 文件到目标机
  for f in ${caFiles[@]}
  do 
    echo "scp $f ${usr}@$h:$f"
    scp $f ${usr}@$h:$f
  done

  echo "Ca files transfered for $h ... ok"

  echo "start install k8s master"

  # --experimental-control-plane  创建一个master实例
  ssh ${h} "${JOIN_CMD} --experimental-control-plane"
done

echo "k8s Cluster create finished."
```

####### 安装 node 节点
安装node节点，则不用copy ca文件，node节点的证书会出master下发。只需要安装 kubeadm 和 kubelet,然后执行一条命令
```shell
# 在master节点执行命令，输出添加node节点的命令
kubeadm token create --print-join-command

# 将上面的输出语句在node节点上执行，节点就添加好了

```

9. 安装 coreDNS 插件
```shell 
# 替换 pod CIDR
curl -fsSL https://raw.githubusercontent.com/839225516/docker_k8s/master/k8s_install_1.14.0_ha/calico.yaml | sed "s#10.172.0.0/16#${CIDR}#g" | kubectl apply -f -
```

10. master node 去污点参考Schedule    
使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。     
这是因为当前的master节点node1被打上了node-role.kubernetes.io/master:NoSchedule的污点：
```shell
kubectl describe node kube-node1 | grep Taint
#Taints:             node-role.kubernetes.io/master:NoSchedule

# 去掉污点
kubectl taint nodes node1 node-role.kubernetes.io/master-
#node "node1" untainted
```

11. 测试DNS
```shell
kubectl run curl --image=radial/busyboxplus:curl -it

nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local


Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

12. kube-proxy开启ipvs    
修改ConfigMap的kube-system/kube-proxy中的config.conf，mode: "ipvs"   
```shell
kubectl edit cm kube-proxy -n kube-system

# 修改完后，重启各个节点的 kube-proxy pod
kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'
```

13. k8s 常用组件安装   

13.1 helm 
```shell
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.12.3-linux-amd64.tar.gz
tar zxvf helm-v2.12.3-linux-amd64.tar.gz 
cd linux-amd64/
cp -p helm /usr/local/bin/
```

helm-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
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

```shell
kubectl apply -f helm-rbac.yaml 

helm init --service-account tiller --tiller-image registry.aliyuncs.com/google_containers/tiller:v2.12.3 \
--stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
#或者：
helm init --upgrade --service-account tiller --tiller-image registry.aliyuncs.com/google_containers/tiller:v2.12.3 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts


helm version
```

13.2  ingress-traefik
```shell
helm search traefik

# 下载package到本地
helm fetch stable/traefik

tar xf traefik-1.24.1.tgz


helm install -n traefik --set dashboard.enabled=true --set dashboard.domain=traefik.jlpay.com \
--set rbac.enabled=true stable/traefik --namespace=kube-system

kubectl get svc traefik --namespace kube-system

# 查看 traefik 添加的ingress访问规则 
kubectl get ingress -n traefik
```

###### 安装 dash-board 1.10
拉取所需镜像和 yaml 文件
```shell 
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0  k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.0/src/deploy/recommended/kubernetes-dashboard.yaml


# 创建一个名为：kubernetes-dashboard-certs的secret
# 使用openssl生成自签名证书

# 1) 生成私钥
openssl genrsa -out dashboard.key 2048

# 2) 生成证书签名请求csr
openssl req -new -out dashboard.csr -key dashboard.key -subj '/O=jlpay.com/CN=k8s.dashboard'

# 3) 生成自签名证书
openssl x509 -req -days 3650 -in dashboard.csr -signkey dashboard.key -out dashboard.crt

# 4) 查看证书
openssl x509 -in dashboard.crt -text -noout

kubectl -n kube-system create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt
```


###### 安装 metrics-server
metrics-server这个容器不能通过CoreDNS 解析各Node的主机名，metrics-server连节点时默认是连接节点的主机名，需要加个参数，让它连接节点的IP，同时因为10250是https端口，连接它时需要提供证书，所以加上–kubelet-insecure-tls，表示不验证客户端证书

https://github.com/kubernetes/kubernetes/tree/release-1.14/cluster/addons/metrics-server（推荐使用此方式）

```shell 
git clone https://github.com/kubernetes-incubator/metrics-server.git


#编辑~/metrics-server/deploy/1.8+/metrics-server-deployment.yaml
      containers:
      - name: metrics-server
        #image: k8s.gcr.io/metrics-server-amd64:v0.3.2
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.2
        imagePullPolicy: IfNotPresent
        command:
        - /metrics-server
        - --kubelet-preferred-address-types=InternalIP
        - --kubelet-insecure-tls

## 在新的版本中，授权文内没有 node/stats 的权限，需要手动去添加
vim resource-reader.yaml 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats  ## 添加此参数
  - namespaces

# 下载镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.2


# 安装 metrics-server
cd metrics-server/deploy/1.8+
kubectl apply -f .


## Metrics API
# 可以通过 kubectl proxy 来访问 Metrics API：
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes/
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/pods
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespace//pods/

# 也可以直接通过 kubectl 命令来访问这些 API，比如:
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes/
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespace//pods/ | jq
```

##### 安装 prometheus-operator
官方github: https://github.com/coreos/prometheus-operator   
这里使用另一个基于官方版本快速部署的： https://github.com/camilb/prometheus-kubernetes.git   

```shell
git clone https://github.com/camilb/prometheus-kubernetes.git
cd  prometheus-kubernetes

# 执行部署脚本, 本地非云环境选择 4) Custom
./deploy

# 删除安装，执行脚本 teardown
```
安装kube-state-metrics时注意镜像下载
```shell
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-state-metrics:v1.5.0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/addon-resizer:1.7
```

####### 如果安装是metrics-server，而不是heapster，要修改prometheus-k8s-service-monitor-kubelet.yaml
```yaml
- port: cadvisor
  interval: 30s
  honorLabels: true 

### to
- path: /metrics/cadvisor
  port: http-metrics
  interval: 30s
  honorLabels: true
```

##### 使用官方github安装 prometheus-operator
使用deployment部署prometheus-operator
```shell
git clone https://github.com/coreos/prometheus-operator.git
kubectl create namespace monitoring
sed -i "s,namespace: default,namespace: monitoring,g" bundle.yaml
kubectl -n monitoring apply -f bundle.yaml
#clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
#clusterrole.rbac.authorization.k8s.io/prometheus-operator created
#deployment.apps/prometheus-operator created
#serviceaccount/prometheus-operator created

kubectl -n monitoring get pods
```

使用operator管理prometheus
####### 创建prometheus-k8s的ServiceAccount，并赋予该账号相应的集群访问权限
prometheus-k8s-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus-k8s
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
```

prometheus-k8s-sts.yaml
```yaml
# 创建prometheus实例
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: k8s
  labels:
    prometheus: k8s
  namespace: monitoring
spec:
  replicas: 2
  version: v2.8.1
  serviceAccountName: prometheus-k8s
  serviceMonitorSelector: 
    matchExpressions:
    - {key: k8s-app, operator: Exists}
  ruleSelector:
    matchLabels:
      role: alert-rules
      prometheus: k8s
  resources:
    requests:
      memory: 1Gi
```

prometheus-k8s-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: k8s
```

###### 使用Operator管理Alertmanager实例
prometheus-k8s-alertmanager.yaml
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: main
  labels:
    alertmanager: main
  namespace: monitoring
spec:
  replicas: 3
  version: v0.16.1
```

由于Prometheus Operator通过Statefulset的方式创建的Alertmanager实例，在默认情况下，会通过alertmanager-{ALERTMANAGER_NAME}的命名规则去查找Secret配置并以文件挂载的方式，将Secret的内容作为配置文件挂载到Alertmanager实例当中。因此，这里还需要为Alertmanager创建相应的配置内容，如下所示，是Alertmanager的配置文件： 
alertmanager.yaml  
```yaml

```