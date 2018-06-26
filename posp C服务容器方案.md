posp C服务容器化方案

最终目标：实现一个容器镜像适用不同环境部署(开发，测试，生产环境)

#### 服务部署

##### docker镜像
采用基础镜像和服务分离的方式：  

    基础镜像不变，包含服务运行所需的所有依赖;  
    更新服务时，每次只需更新服务程序  

base_c dockerfile
```dockerfile
FROM 172.20.8.199:5001/library/centos:7
MAINTAINER posp base_c image centos7.4

# Base Install:时区、字符、vim、net-tools
RUN rpm --rebuilddb \
    && yum -y install \
       vim \
       tar \
       net-tools \
       telnet \
       gdb  \
       file \
    && rm -rf /var/cache/yum/* \
    && yum clean all  \
    &&  ln -sf /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime \
    && mkdir /data/

# 环境变量
ENV LANG en_US.UTF-8
ENV LD_LIBRARY_PATH  $LD_LIBRARY_PATH:/data/lib:/data/oracle/lib
ENV NLS_LANG  AMERICAN_AMERICA.AL32UTF8
ENV TNS_ADMIN  /data/oracle/network/admin
# /data/oracle/network/admin 配置oracle db 

# 导入服务名
# ENV PROC {service_name}

# 添加c服务的依赖包
ADD lib_c.tar.gz /data/
#COPY bin/PROC /data/bin/
#COPY etc/PROC.ini /data/etc/

WORKDIR /data/bin/

CMD ["/bin/bash","/data/bin/start.sh"]
```

服务启动脚本start.sh
```shell
#!/bin/bash
# start posp c progress
# 容器启动时，将服务名通过环境变量PROC导入
# RUNENV 变量用于取不同环境的配置文件

set -eu 

# mv git file to bin/
cp /tmp/${GIT_DIR}/${PROC} /data/bin/
if [ "$RUNENV" = "test" ]; then
    cp /tmp/${GIT_DIR}/${PROC}.ini.${RUNENV}  /data/etc/${PROC}.ini
elif [ "$RUNENV" = "pre" ]; then
    cp /tmp/${GIT_DIR}/${PROC}.ini.${RUNENV}  /data/etc/${PROC}.ini
else
    cp /tmp/${GIT_DIR}/${PROC}.ini /data/etc/${PROC}.ini
fi

sleep 2

# touch log_file
if [ ! -f /data/log/${PROC}.log ];then
    touch /data/log/${PROC}.log
fi 

if [ -n "${PROC}" ];then
    if [ -f ${PROC} ];then

        if [ ! -x ${PROC} ];then
            chmod +x ${PROC}
        fi

        echo start ... ${PROC}
        ./${PROC}  ../etc/${PROC}.ini &
        sleep 2
    fi

        #tail -f /data/log/${PROC}.log

        while true ;do
                ps -ef|grep "${PROC}"|grep -v "grep"
                if [ "$?" == "1" ];then
                        echo "service down"
                        kill 1
                        exit
                fi
                sleep 8

        done

fi
```
##### 拉取服务程序
服务程序通过k8s的gitRepo volume,从git代码仓库clone到容器  
gitRepo volume 简介：
    
    gitRepo volume会挂载到一个空目录中，并clone一份指定的git代码到该目录中
    gitRepo volume的生命周期与pod同步
    clone操作会先于pod创建操作。也就是说pod创建后，代码已在挂载的目录中

k8s的deployment yaml文件
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: posp-{{ .Release.Name }}
spec: 
  replicas: 1
  template:
    metadata:
      labels:
        name: {{ .Values.containers.name }} 
    spec:
      containers:
      - name: {{ .Values.containers.name }}
        image: {{ .Values.containers.image }}
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: git-volume
          mountPath: /tmp
        - name: app-logs
          mountPath: /data/log
        env:
        - name: RUNENV
          value: {{ .Values.runenv }}
        - name: PROC
          value: {{ .Values.configmap.proc }}
        - name: GIT_DIR
          value: {{ .Values.configmap.project }}
      - name: filebeat
        image: {{ .Values.filebeat.image }}
        imagePullPolicy: IfNotPresent 
        volumeMounts:
        - name: app-logs
          mountPath: /log
        - name: filebeat-config
          mountPath: /etc/filebeat/
      volumes:
      - name: app-logs
        emptyDir: {}
      - name: filebeat-config
        configMap:
          name: {{ .Release.Name }}-filebeat-config
      - name: git-volume
        gitRepo:
          repository:  {{ .Values.git.project }}
          revision: {{ .Values.git.revision }}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-filebeat-config
data:
  filebeat.yml: |
    filebeat.prospectors:
    - type: log
      enable: true
      paths:
        - /log/*.log
      multiline.pattern: ^\[
      multiline.negate: true
      multiline.match: after
      tags: {{ .Values.filebeat.tags }}

    output.redis:
      enable: true
      hosts: "10.0.116.62"
      port: 6379
      key: "logstash:redis"
      datatype: list
```

这些变量由helm的valus.yaml文件管理：  
values.yaml
```yaml
containers:
  name: account-confirm
  image: 10.0.116.41:80/posp-verify/base_c:accp_record
configmap:
  proc: account_confirm
  project: account_confirm
nfs:
  server: 10.0.116.71
  mdir: "/data/log"
filebeat:
  image: 10.0.116.41:80/library/filebeat:latest
  tags: posp_trade_c
git:
  project: "http://172.20.5.54/posp/posp_c/account_confirm.git"
  revision: "ecc71b65f2f09b2a044dff8d7f12d777e052b8b3"
runenv: "pre"
```

##### 制作dock镜像及helm charts
1)docker镜像
```shell
# tree c_base/ -L 2
c_base/
├── dockerfile
├── lib_c.tar.gz
└── posp_lib
    ├── bin
    ├── etc
    ├── lib
    ├── log
    └── oracle

# cd posp_lib 
# tar zcpf lib_c.tar.gz bin etc lib log oracle
# mv lib_c.tar.gz ../
# docker build -t 172.20.8.199:5001/posp-verify/base_c:posp_elk .
# docker push 172.20.8.199:5001/posp-verify/base_c:posp_elk
```

制作helm charts
``` shell
# helm create accp-finance          #生成accp-finance目录,并生产chart模板
accp-finance/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml

只保留需要的文件，并修改Chart.yaml templates/pod.yaml values.yaml 文件
accp-finance/
├── charts
├── Chart.yaml
├── templates
│   └── pod.yaml
└── values.yaml
```
#### 配置管理
配置文件管理  
配置文件统一放在gitlab仓库，以不同后缀名区分不同环境的配置，和服务程序一同git clone到容器  
通过启动时设置的启动环境，选择对应的配置文件：

eg: 

    accp_record.ini             生产环境(product)的配置文件
    accp_record.ini.pre         生产验证环境(pre)的配置文件
    accp_record.ini.test        测试环境（test）的配置文件

#### 日志管理
服务的日志通过filebeat，抽取到elasticstash存储

