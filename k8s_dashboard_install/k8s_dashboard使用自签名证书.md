##### 使用openssl生成自签证书
由于默认安装自动生成的证书是 0001年1月签发的，早已过期，只能使用firefox浏览器访问   


##### 拉取所需镜像和 yaml 文件
```shell 
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0  k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.0/src/deploy/recommended/kubernetes-dashboard.yaml
```

将生成证书那段yaml注释掉
```yaml 
# ------------------- Dashboard Secret ------------------- #
# 这段注释掉
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
```

这里使用自签证书替换掉：
```shell 
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

# 5) 创建secret
kubectl -n kube-system create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt
```




