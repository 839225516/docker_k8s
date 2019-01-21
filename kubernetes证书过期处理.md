##### 查看证书是否过期
```shell
openssl x509 -in /etc/kubernetes/ssl/kubelet-client.crt -noout -dates
```

##### kubelet证书轮换
kubernetes 1.8以上版本支持kubelet证书轮换，在当前证书即将过期时， 将自动生成新的秘钥，并从 Kubernetes API 申请新的证书。   

当 kubelet 启动时，如被配置为自举（使用--bootstrap-kubeconfig 参数），  
kubelet 会使用其初始证书连接到 Kubernetes API ，并发送证书签名的请求。   
可以通过以下方式查看证书签名请求的状态：
```shell 
kubectl get csr
```
最初，来自节点上 kubelet 的证书签名请求处于 Pending 状态。   
如果证书签名请求满足特定条件， controller manager控制器管理器会自动批准，此时请求会处于 Approved 状态。   
接下来，控制器管理器会签署证书， 证书的有效期限由 --experimental-cluster-signing-duration 参数指定，签署的证书会被附加到证书签名请求中。   
Kubelet 会从 Kubernetes API 取回签署的证书，并将其写入磁盘，存储位置通过 --cert-dir 参数指定。 然后 kubelet 会使用新的证书连接到 Kubernetes API。
当签署的证书即将到期时，kubelet 会使用 Kubernetes API，发起新的证书签名请求。

##### 配置kubelet证书轮换
1. controller-manager 参数    
--feature-gates=RotateKubeletServerCertificate=true
--experimental-cluster-signing-duration 该参数控制证书签发的有效期限  
```conf
--feature-gates=RotateKubeletServerCertificate=true
--experimental-cluster-signing-duration=87600h0m0s
```

2. kubelet 参数
```conf
--feature-gates=RotateKubeletServerCertificate=true
```

3.创建 rbac对象
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests/selfnodeserver
  verbs:
  - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeadm:node-autoapprove-certificate-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
```


