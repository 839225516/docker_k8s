#### 安装 gitlab-runner 并使用kubernetes执行器（Executor）
这里的重点是讲 kubernetes Executor,有两种架构一种是gitlab-runner 安装在kvm的虚拟机，Executor为 kubernetes 

```conf
                                       +----------------------------+
                                       | kubernetes Cluster         |
                                       |                            |
                                       |       +-------+            |
                                       |       |Job 1  |Pod         |
                                    +------->  |       |            |
+------+           +-------------+     |       +-------+            |
|gitlab| --------> |gitlab runner|     |                            |
+------+           +-------------+     |       +-------+            |
                                    +------->  |Job 2  |Pod         |
                                       |       |       |            |
                                       |       +-------+            |
                                       +----------------------------+
```

另一种是将gitlab-runner 也部署在k8s中 
```conf
                       +---------------------------------------+
                       | Kubernetes Cluster                    |
                       |                                Pod    |
                       |                             +-------+ |
                       |                       +---> | Job 1 | |
+--------+             | +-------------+ +---+/      +-------+ |
| Gitlab | +---------> | |Gitlab Runner|                       |
+--------+   Trigger   | +-------------+ +---+\      +-------+ |
             Runner    |       Pod             +---> | Job 2 | |
                       |                             +-------+ |
                       |                                Pod    |
                       |                                       |
                       +---------------------------------------+ 
```


gitlab 的 .gitlab-ci.yml 文件定义job, 传给gitlab-runner， gitlab-runner配置的Excutor为k8s,通过kube-apiserver的https接口启动对应images的Pod,在pod上完成相应的job    
gitlab-runner只是下发相关job给 Executors 
runner 要访问k8s的api接口，需要配置k8s的ca证书 ca.crt, 还要k8s有相应rbac权限的SERVICE_ACCOUNT的token才能和k8s集群通信

##### 查看k8s集群的ca.crt
```shell
# 查看k8s集群的ca.key
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d
```
##### 在k8s集群创建有创建pod权限的 ServiceAccount
ServiceAccount name: runnner-executor
namespace: gitlab

gitlab-runner-rbac.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: runner-executor
  namespace: gitlab
#imagePullSecrets:
#- name: dockersecret
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: gitlab
  name: runner-executor
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: gitlab
  name: runner-executor-rolebinding
subjects:
- kind: ServiceAccount
  name: runner-executor
  namespace: gitlab
roleRef:
  kind: Role
  name: runner-executor
  apiGroup: rbac.authorization.k8s.io
```

查看 namespace: gitlab 下 serviceaccoutname: runner-executor的token
```shell
# 查看 token , 修改 SERVICE_ACCOUNT_NAME 为 runner-executor
kubectl get secrets -o \
jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='SERVICE_ACCOUNT_NAME')].data.token}" \
-n gitlab | base64 -d
```

##### gitlab 上查看ci/cd 注册token
http://172.20.5.54/    ZyGTTTmNwshnAbCF8jm5


##### gitlab-runner 注册
```shell
gitlab-runner register \
  --non-interactive \
  --url "GITLAB_URL" \
  --registration-token "GITLAB_CI_TOKEN" \
  --executor "kubernetes" \
  --description "kubernetes-runner" \
  --tag-list "k8s-runner" \
  --kubernetes-host "https://KUBERNETES_URL" \
  --kubernetes-ca-file "/etc/k8s/ca.crt" \
  --kubernetes-bearer_token "SERVICE_ACCOUNT_BEARER_TOKEN"
```

或者直接部署在k8s中     
deployment-runner.yaml
```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: runner
  namespace: gitlab
  labels:
    app: runner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: runner
  template:
    metadata:
      labels:
        app: runner
    spec:
      containers:
      - name: ci-builder
        image: gitlab/gitlab-runner:latest
        command:
        # 命令有点长，做了以下几步：注销当前的 runner name 以防止 runner 冲突；注册新的 runner；启动 runner daemon
        - /bin/bash
        - -c
        - "/usr/bin/gitlab-runner unregister -n $RUNNER_NAME || true; /usr/bin/gitlab-runner register; exec /usr/bin/gitlab-runner run"
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/ssl
          #subPath: ssl
          name: ca-crt
        env:
        - name: RUNNER_NAME
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        envFrom:
        # 通过 ConfigMap 注入 runner 配置
        - configMapRef:
            name: gitlab-runner-cfg
        # gitlab-runner 自带 Prometheus metrics server，通过上面的 METRICS_SERVER 环境变量配置
        # 强的一比！
        ports:
        - containerPort: 9100
          name: http-metrics
          protocol: TCP
        lifecycle:
          # 在 pod 停止前，注销这个 runner
          preStop:
            exec:
              command:
              - /bin/bash
              - -c
              - "/usr/bin/gitlab-runner unregister -n $RUNNER_NAME"
      restartPolicy: Always
      volumes: 
      - name: ca-crt
        configMap:
          name: cm-ca-crt
```

configmap-runner.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: gitlab
  labels:
    app: gitlab-deployer
  name: gitlab-runner-cfg
data:
  # 具体可用的参数配置以及环境变量配置可以运行 gitlab-runner register --help 查看
  REGISTER_NON_INTERACTIVE: "true"
  REGISTER_LOCKED: "false"
  CI_SERVER_URL: "http://172.20.5.54/"
  REGISTRATION_TOKEN: "FoN6VUuscn2tfbHJwdsada"
  METRICS_SERVER: "0.0.0.0:9100"
  RUNNER_CONCURRENT_BUILDS: "4"
  RUNNER_REQUEST_CONCURRENCY: "4"
  RUNNER_TAG_LIST: "k8s-runner"
  RUNNER_EXECUTOR: "kubernetes"
  KUBERNETES_NAMESPACE: "gitlab"
  KUBERNETES_PULL_POLICY: "if-not-present"
  KUBERNETES_HELPER_IMAGE: "gitlab/gitlab-runner-helper:x86_64-latest"
  DOCKER_HELPER_IMAGE: "gitlab/gitlab-runner-helper:x86_64-latest"
  KUBERNETES_HOST: "https://172.20.2.225:6443"
  KUBERNETES_CA_FILE: "/etc/ssl/ca.crt"
  KUBERNETES_SERVICE_ACCOUNT: "runner-executor"
  KUBERNETES_BEARER_TOKEN: "runner-executor的token值"
```

** 特别注意 **   
gitlab-runner的executor在使用docker或者kubernetes时，在执行ci/cd时，会在docker中先拉取 gitlab-runner-helper 镜像，来帮忙git clone 仓库。



