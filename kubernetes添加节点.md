新装系统 CentOS 7  
初始化系统  

#### 升级内核
```shell 
yum --enablerepo=elrepo-kernel install kernel-ml-devel kernel-ml

yum   install  kernel-ml-devel kernel-ml
```

升级后切换内核：
```shell 
vim /etc/default/grub
GRUB_DEFAULT=0

grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 安装docker-ce
> yum install -y docker-ce

##### 配置docker启动项
``` shell 
mkdir -p /usr/lib/systemd/system/docker.service.d/

vim /usr/lib/systemd/system/docker.service.d/docker-options.conf
[Service]
Environment="DOCKER_OPTS=--insecure-registry=http://10.0.116.41:80 --graph=/opt/docker --registry-mirror=http://b438f72b.m.daocloud.io --disable-legacy-registry"

vim /usr/lib/systemd/system/docker.service.d/docker-dns.conf
[Service]
Environment="DOCKER_DNS_OPTIONS=\
    --dns 172.200.0.2 --dns 114.114.114.114  \
    --dns-search default.svc.cluster.local --dns-search svc.cluster.local  \
    --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2  \
```

##### 将iptables FORWARD链的默认策略修改为ACCEPT
安装 docker 之后，会将iptables的FORWARD链设置为DROP，需要修改为ACCEPT
```shell
iptables -P FORWARD ACCEPT
iptables-save > /etc/sysconfig/iptables

systemctl daemon-reload
systemctl start docker
systemctl enable docker
systemctl status docker
```

#### k8s安装

##### 复制k8s所需证书
从老节点复制证书
```shell 
mkdir -pv /etc/kubernetes/ssl/
scp 10.0.116.72:/etc/kubernetes/ssl/* /etc/kubernetes/ssl/
```

##### 复制k8s应用程序
> scp -r 10.0.116.72:/usr/local/bin/{calicoctl,kubectl,kubefed,kubelet,kube-proxy} /usr/local/bin/

##### 配置 kubelet和kube-proxy
kubelet 
```shell
# 首先 创建 kubelet kubeconfig 文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证
kubectl config set-credentials kubelet-bootstrap \
  --token=4caab586aa3a2c344c096d46597be149 \
  --kubeconfig=bootstrap.kubeconfig

# 配置关联
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 生成文件，配置默认关联
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件
mv bootstrap.kubeconfig /etc/kubernetes/
```

kube-proxy
```shell
# 创建kube-proxy kubeconfig 文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-proxy.kubeconfig

# 配置客户端认证
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

# 配置关联
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

# 配置默认关联
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到目录
mv kube-proxy.kubeconfig /etc/kubernetes/
``` 

配置kubelet  kube-proxy 服务文件kubelet.service
mkdir -p /data/kubelet  
vim /etc/systemd/system/kubelet.service
```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/data/kubelet
ExecStart=/usr/local/bin/kubelet \
  --cgroup-driver=cgroupfs \
  --hostname-override=10.0.116.42 \
  --pod-infra-container-image=10.0.116.41:80/k8s/pod-infrastructure:latest \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=172.200.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --fail-swap-on=false \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --network-plugin=cni \
  --network-plugin-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/cni/bin \
  --v=2

[Install]
WantedBy=multi-user.target
```

mkdir -p /data/kube-proxy   
vim /etc/systemd/system/kube-proxy.service
```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/data/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=10.0.116.42 \
  --hostname-override=10.0.116.42 \
  --cluster-cidr=172.200.0.0/16 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

#### 创建Nginx 代理
在每个 node 都必须创建一个 Nginx 代理， 这里特别注意， 当 Master 也做为 Node 的时候 不需要配置 Nginx-proxy
``` shell
mkdir -p /etc/nginx

# 写入代理配置
cat << EOF >> /etc/nginx/nginx.conf
error_log stderr notice;

worker_processes auto;
events {
  multi_accept on;
  use epoll;
  worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 10.0.116.71:6443;
        server 10.0.116.72:6443;
        server 10.0.116.73:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

# 更新权限
chmod +r /etc/nginx/nginx.conf

#配置 Nginx 基于 docker 进程，然后配置 systemd 来启动
cat << EOF >> /etc/systemd/system/nginx-proxy.service
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \
                              -v /etc/nginx:/etc/nginx \
                              --name nginx-proxy \
                              --net=host \
                              --restart=on-failure:5 \
                              --memory=512M \
                              10.0.116.41:80/library/nginx:alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF

# 启动 Nginx
systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
systemctl status nginx-proxy


# 启动 Node 的 kubelet 与 kube-proxy
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

在master节点执行
```shell
kubectl get csr
# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-6-7LtqmlyY5f6Z-EyDAOwYbmWUNgU9wmMs8Aw92Y5QM   2m        kubelet-bootstrap   Pending
node-csr-CmISs64_UmrORBMC27vUu7igB4oLlnQpsoCZrJzVAQM   162d      kubelet-bootstrap   Approved,Issued
node-csr-H6iDcUFsbnhKzC3nxUvat3uIYRgjTWmGzydy5ukPF4k   109d      kubelet-bootstrap   Approved,Issued
node-csr-IwmxVWTHvX1cUWOkdYxlNHeFw2P3fAANSKU5988s7EE   45d       kubelet-bootstrap   Approved,Issued
node-csr-hY5n8EIbiQrFy1pXcvBPNVX5DomzRmF_qnOqiMAmyEw   109d      kubelet-bootstrap   Approved,Issued
node-csr-kYw1B0WUOb1Mm802EvVkMAzin-wp7NqBJHmtoJnOzSw   162d      kubelet-bootstrap   Approved,Issued
node-csr-ohGkBy1bswOr6YnvFpemcV4X0G-ZhoAZci_PTIjN6lM   162d      kubelet-bootstrap   Approved,Issued

kubectl certificate approve {NAME}
# kubectl certificate approve node-csr-6-7LtqmlyY5f6Z-EyDAOwYbmWUNgU9wmMs8Aw92Y5QM

kubectl get nodes
```

#### 将k8s的node节点设置不可以使用
暂时不能让生成的pod在此node上运行，需要通知kubernetes让其不要创建过来，这条命令就是cordon，uncordon则是取消这个设置
```shell
查看 node NAME
# kubectl get nodes

根据node-NAME 将node 设置为不可以使用
# kubectl cordon {node-NAME}

将node 设置为可以使用
# kubectl uncordon {node-NAME}
# 
```


#### k8s node 的维护模式
维护模式是将 node 设置为cordon状态，并在其它node重新创建本节点的pod，evict(回收)本节点的pod
> kubectl drain {node_NAME}
