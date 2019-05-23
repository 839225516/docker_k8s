kubernetes手动安装集群升级kubeadm安装

升级目标：
1) 集群升级为kubeadm安装方式，方便后续升级；
2) 统一docker版本和配置  
    版本：yum install -y --setopt=obsoletes=0  docker-ce-18.06.1.ce-3.el7   
    storage driver: overlay2  
    数据目录： /data/docker/  
    默认不添加iptables FORWARD链的drop策略  
3) kubernetes集群配置： 
``` conf
    kubelet:   
      统一pod的数据目录为/data/kubelet
    kube-apiserver: 
      新增域名负载均衡：api.k8s.jlpay.io  
      禁用匿名访问
    kube-controller-manager:
      开启kubelet的server证书自动续期
      开启hpa参数   

    heapster 升级为 metrics-server
```
4) kubernetes 升级到1.14 （看升级进度）


升级方式：  
1) 备份etcd的snapshot
2) 先升级其中一个master节点
3) 新增一个node节点
4）升级其它的master和node节点

一. 备份etcd （注意v2和v3数据）及恢复etcd操作
```shell
ETCDCTL_API=3 etcdctl --endpoints="https://172.20.2.236:2379,https://172.20.2.237:2379,https://172.20.2.238:2379" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" --cacert="/etc/kubernetes/ssl/ca.pem"  snapshot save etcdsnapshot_$(date +%F).db


# 用snapshot恢复集群
# 首先需要分别停掉三台Master机器的kube-apiserver，确保kube-apiserver已经停止了

# 停止 etcd 服务，并删除数据目录的文件
systemctl stop etcd
mv /var/lib/etcd /var/lib/etcd.bak.$(date +%F)

# 分别在每台etcd服务器上执行
ETCDCTL_API=3 etcdctl snapshot restore etcdsnapshot_2019-05-06.db \
    --endpoints="https://172.20.2.236:2379" \
    --name=etcd1 \
    --cert="/etc/etcd/ssl/etcd.pem" \
    --key="/etc/etcd/ssl/etcd-key.pem" \
    --cacert="/etc/kubernetes/ssl/ca.pem" \
    --initial-advertise-peer-urls="https://172.20.2.236:2380" \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster="etcd1=https://172.20.2.236:2380,etcd2=https://172.20.2.237:2380,etcd3=https://172.20.2.238:2380" \
    --data-dir=/var/lib/etcd

# 在etcd2执行
ETCDCTL_API=3 etcdctl snapshot restore etcdsnapshot_2019-05-06.db \
    --endpoints="https://172.20.2.237:2379" \
    --name=etcd2 \
    --cert="/etc/etcd/ssl/etcd.pem" \
    --key="/etc/etcd/ssl/etcd-key.pem" \
    --cacert="/etc/kubernetes/ssl/ca.pem" \
    --initial-advertise-peer-urls="https://172.20.2.237:2380" \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster="etcd1=https://172.20.2.236:2380,etcd2=https://172.20.2.237:2380,etcd3=https://172.20.2.238:2380" \
    --data-dir=/var/lib/etcd

# 在etcd3上执行
ETCDCTL_API=3 etcdctl snapshot restore etcdsnapshot_2019-05-06.db \
    --endpoints="https://172.20.2.238:2379" \
    --name=etcd3 \
    --cert="/etc/etcd/ssl/etcd.pem" \
    --key="/etc/etcd/ssl/etcd-key.pem" \
    --cacert="/etc/kubernetes/ssl/ca.pem" \
    --initial-advertise-peer-urls="https://172.20.2.238:2380" \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster="etcd1=https://172.20.2.236:2380,etcd2=https://172.20.2.237:2380,etcd3=https://172.20.2.238:2380" \
    --data-dir=/var/lib/etcd

# 启动etcd，启动kube-apiserver
```

二.升级准备工作
1. 生成k8s集群证书
1) ca证书使用原集群老证书
``` shell
# 原证书 ca.pem ca-key.pem
mkdir -p /etc/kubernetes/pki/
cp -p ca.pem /etc/kubernetes/pki/ca.crt && cp -p ca-key.pem /etc/kubernetes/pki/ca.key
```
2) etcd client证书 apiserver-etcd-client.crt,apiserver-etcd-client.key
```shell
# 添加ca配置文件 etcd-config.json
cd /etc/kubernetes/pki/
cat > etcd-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "server": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

# 添加 apiserver-etcd-client 的证书签名请求文件 apiserver-etcd-client.json
cat > apiserver-etcd-client.json << EOF
{
    "CN": "client",
    "hosts": [
       ""
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
           "C": "CN",
           "ST": "HangZhou",
           "L": "XS",
           "O": "k8s",
           "OU": "System"
        }
    ]
}
EOF

# 这里只要生成etcd的client证书
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=etcd-config.json \
-profile=client  apiserver-etcd-client.json | cfssljson -bare apiserver-etcd-client
mv apiserver-etcd-client-key.pem apiserver-etcd-client.key \
&& mv apiserver-etcd-client.pem apiserver-etcd-client.crt
```

3) kube-apiserver服务端证书
```shell
# 证书配置文件 config.json
cat > config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

# apiserver证书配置文件
# master节点为 172.20.2.236，172.20.2.237，172.20.2.238
# apiserver ClusterIP 为 10.254.0.1

cat > apiserver.json << EOF
cat > apiserver.json << EOF
{
    "CN": "kube-apiserver",
    "hosts": [
      "172.20.2.236",
      "172.20.2.237",
      "172.20.2.238",
      "api.k8s.jlpay.io",      
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"     
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    }
}
EOF

cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes  apiserver.json | \
cfssljson -bare apiserver && mv apiserver-key.pem apiserver.key && mv apiserver.pem apiserver.crt
```

4) 生成kubelet客户端证书:apiserver-kubelet-client.key,apiserver-kubelet-client.crt
```shell
cat > apiserver-kubelet-client.json << EOF
{
  "CN": "kube-apiserver-kubelet-client",
  "hosts": [""],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
EOF

cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes \
apiserver-kubelet-client.json | cfssljson -bare apiserver-kubelet-client \
&& mv apiserver-kubelet-client-key.pem apiserver-kubelet-client.key \
&& mv apiserver-kubelet-client.pem apiserver-kubelet-client.crt
```

5) 生成kube-apiserver 代理证书：front-proxy-ca.crt front-proxy-ca.key front-proxy-client.crt front-proxy-client.key
```shell
cat > front-proxy-ca.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}

cfssl gencert -initca front-proxy-ca.json | cfssljson -bare front-proxy-ca  && mv front-proxy-ca.pem front-proxy-ca.crt && mv front-proxy-ca-key.pem front-proxy-ca.key

cat > front-proxy-client.json << EOF
{
  "CN": "front-proxy-client",
  "hosts": [""],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -ca=front-proxy-ca.crt -ca-key=front-proxy-ca.key -config=config.json \
-profile=kubernetes  front-proxy-client.json | cfssljson -bare front-proxy-client \
&& mv front-proxy-client-key.pem front-proxy-client.key \
&& mv front-proxy-client.pem front-proxy-client.crt
```

6) Service Account秘钥：给kube-controller-manager使用
```shell
openssl genrsa -out sa.key 1024  &&   openssl rsa -in sa.key -pubout -out sa.pub
```

###### 证书文件
```conf
|-- apiserver.crt
|-- apiserver-etcd-client.crt
|-- apiserver-etcd-client.key
|-- apiserver.key
|-- apiserver-kubelet-client.crt
|-- apiserver-kubelet-client.key
|-- ca.crt
|-- ca.key
|-- front-proxy-ca.crt
|-- front-proxy-ca.key
|-- front-proxy-client.crt
|-- front-proxy-client.key
|-- sa.key
|-- sa.pub
```

2. kubeadm的配置文件 : kubeadm-config.yaml
```yaml
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.6
imageRepository: registry.jlpay.io/google_containers
api:
  advertiseAddress: 172.20.2.236
  bindPort: 6443
  controlPlaneEndpoint: api.k8s.jlpay.io
apiServerCertSANs:
- 172.20.2.236 
- 172.20.2.237
- 172.20.2.238
- api.k8s.jlpay.io
- 10.254.0.1
networking:
  # This CIDR is a Calico default. Substitute or remove for your CNI provider.
  serviceSubnet: 10.254.0.0/16
  podSubnet: 192.168.0.0/16
etcd:
  external:
    caFile: /etc/kubernetes/pki/etcd/ca.crt
    certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
    keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
    endpoints:
    - https://172.20.2.236:2379
    - https://172.20.2.237:2379
    - https://172.20.2.238:2379
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: "true"
  horizontal-pod-autoscaler-sync-period: 10s
  feature-gates: "RotateKubeletServerCertificate=true"
kind: MasterConfiguration
kubeProxy:
  config:
    mode: "iptables"
    #portRange: "30000-35000"
kubeletConfiguration:
  baseConfig:
    clusterDNS:
    - 10.254.0.2
```

3. 系统初始化    
```shell
# 1) 修改主机名：   
hostnamectl set-hostname kube-master-172.20.2.236

# 2) 添加hosts
cat <<EOF >> /etc/hosts
172.20.20.247        kube-master
172.20.20.248        kube-node1
172.20.20.249        kube-node2
172.20.2.236         kube-master-172.20.2.236
EOF

# 3) 关闭 firewalld, selinux, swap 
systemctl stop firewalld && systemctl disable firewalld

setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux

swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab

# 4) 创建/etc/sysctl.d/k8s.conf文件，添加如下内容
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 执行命令使修改生效
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge

# 5) 安装docker 
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install -y --setopt=obsoletes=0 \
  docker-ce-18.06.1.ce-3.el7

cat > /etc/docker/daemon.json << EOF
{
    "icc": true,
    "insecure-registries": ["registry.jlpay.io"],
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "graph": "/data/docker"
}
EOF


systemctl start docker
systemctl enable docker

# 6) 安装kubeadm, kubelet, kubectl
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install kubeadm-1.11.6 kubelet-1.11.6 kubectl-1.11.6

systemctl start kubelet
systemctl enable kubelet
```

4. 使用kubeadm安装k8s集群

安装步骤： 

    将node节点改成,不可调度状态  kubectl cordon {NODE-NAME}
    先升级其中一个master节点，升级成功后再升级其它的node节点

####### 注意，使用kubeadm安装k8s集群会使 kube-proxy 以static-pod的方式启动，当master安装好后，其它的节点也会自动启动kube-proxy,可以将其它节点的kube-proxy服务先停了，停kube-proxy对老的pod没有影响

##### 安装master节点
先将前面生成的证书文件上到目录 /etc/kubernetes/pki/  
``` shell
# 1) 下载镜像
kubeadm --config kubeadm-config.yaml config images pull

# 2) 修改tag为google官方镜像, 这步在node节点上要操作
kubeadm --config kubeadm-config.yaml config images list | awk -F / '{system("docker tag "$0" k8s.gcr.io/"$NF) }'

# 上传到私有镜像
# kubeadm --config kubeadm-config-test.yaml config images list | awk -F / '{system("docker tag "$0" registry.jlpay.io/google_containers/"$NF "&& docker push registry.jlpay.io/google_containers/"$NF ) }'

# 3) 检查节点是否符合要求,这步可以检查哪项没有达到安装条件（如果条件都满足，这步会直接完成master安装）
# kubeadm init phase preflight 顺利的话，这后面的步骤不用执行
kubeadm init phase preflight --config kubeadm-config.yaml

# 4) 生成证书，这步骤不用执行了，手动生成证书
# kubeadm init phase certs --config kubeadm-config.yaml

# 5) 生成kubelet 、scheduler 、controller-manager 、admin的kubeconfig文件
# kubeadm init phase kubeconfig all --config kubeadm-config.yaml

# 6) 生成kubelet的配置文件，并启动kubelet
# kubeadm init phase kubelet-start --config kubeadm-config.yaml

# 7) 创建apiserver,controller-manager,scheduler 的static Pod的manifest yaml文件
# kubeadm init phase control-plane --config kubeadm-config.yaml

# 8) 启动etcd pod,这一步也不用做了，用外部的etcd集群
# kubeadm init phase etcd --config kubeadm-config.yaml

# 9) 上传证书到集群
# kubeadm init phase upload-certs --config kubeadm-config.yaml

# 10) 将当前节点标记为master,并打上污点
# kubeadm init phase mark-control-plane --config kubeadm-config.yaml

# 11) 生成bootstrap-token 用于添加node节点
# kubeadm init phase bootstrap-token --config kubeadm-config.yaml

# 12) 上传kubeadm的配置文件到集群的configmap
# kubeadm init phase upload-config [all|kubeadm|kubelet] --config kubeadm-config.yaml

# 13) 安装其它的组件，kube-proxy , coredns
# kubeadm init phase addon [all|kube-proxy|coredns] --config kubeadm-config.yaml

# 安装完master节点后，如果要去除master污点，执行下面命令
kubectl taint nodes NODENAME node-role.kubernetes.io/master-
```

安装完一个节点后，先验证集群是否正常，coredns是否能正常解析，正常后，继续升级下一个节点或master     

    查看kube-apiserver, kube-controller-manger, kube-scheduler, kube-proxy的日志，看是否有异常
    验证coredns是否正常


##### 升级中存在的问题
1) kube-proxy问题：  
   使用kubeadm安装k8s集群会使 kube-proxy 以static-pod的方式启动，当master安装好后，其它的节点也会自动启动kube-proxy,可以将其它节点的kube-proxy服务先停了，停kube-proxy对老的pod没有影响

2) coredns问题：
   老版本中coredns的ClusterIP 是*.*.*.2; 新版本默认是*.*.*.10; 有两种解决方式，一种是重新安装coredns，并把kubelet的dns配置项 --cluster_dns 改成*.*.*.10   
   二是kubeadm配置中修改clusterDNS:10.254.0.2,注意这里也有一个问题，如果要重新安装的话要手动生成svc或修改svc:  
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: KubeDNS
  name: kube-dns
  namespace: kube-system
  selfLink: /api/v1/namespaces/kube-system/services/kube-dns
spec:
  clusterIP: 10.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  selector:
    k8s-app: kube-dns
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

###### kubeadm 升级kubernetes集群版本
``` shell
# 查询可用的版本
yum list --showduplicates | grep 'kubeadm\|kubectl\|kubelet'

yum update kubeadm-1.12.6-0 kubectl-1.12.6-0 kubelet-1.12.6-0

kubeadm upgrade apply --config=kubeadm-config-test.yaml

#使用 kubectl version 来查看状态
kubectl version

#使用 kubectl cluster-info 查看服务地址
kubectl cluster-info

# 最后重启 kubelet 
systemctl daemon-reload
systemctl restart kubelet

```
##### Node 节点升级
升级对应的 kubelet kubeadm kubectl 的版本，拉取对应版本的镜像即可