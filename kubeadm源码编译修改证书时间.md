##### kubeadm 源码编译修改证书时间
###### 安装编译环境 golang
```shell
wget https://studygolang.com/dl/golang/go1.12.1.linux-amd64.tar.gz
tar zxf go1.12.1.linux-amd64.tar.gz -C /usr/local/
```

编辑/etc/profile文件添加golang环境变量：    
vim /etc/profile
```conf
#go setting
export GOROOT=/usr/local/go
export GOPATH=/data/gopath
export PATH=$PATH:$GOROOT/bin
```

之后，执行 source /etc/profile 使其生效


###### 下载 kubernetes，并修改源码的证书时间为 10年
```shell
# 拉取源码
cd /data && git clone https://github.com/kubernetes/kubernetes.git

# 切换到 1.14.0版本
git checkout -b remotes/origin/v1.14.0 v1.14.0
```

修改源码   /data/kubernetes/staging/src/k8s.io/client-go/util/cert/cert.go      
vim  /data/kubernetes/staging/src/k8s.io/client-go/util/cert/cert.go
```golang
NotAfter:     time.Now().Add(duration365d * 10).UTC(),

NotAfter:  validFrom.Add(maxAge *10),

NotAfter:  validFrom.Add(maxAge * 10),
```

编译
```shell
cd /data/kubernetes/ && make WHAT=cmd/kubeadm

# 查看编译后的文件
ls -l /data/kubernetes/_output/bin/kubeadm

# 替换 kubeadm
mv /usr/bin/kubeadm /usr/bin/kubeadm_backup
ln -s /data/kubernetes/_output/bin/kubeadm /usr/bin/kubeadm
```

###### 查看证书的期限
```shell
openssl x509 -in front-proxy-client.crt   -noout -text  |grep Not
```
