## docker DNS配置  ##

####  容器的dns解析顺序 #####
容器中的DNS名称解析优先级顺序为：
    
    内置DNS服务器127.0.0.11。
    通过--dns等参数为容器配置的DNS服务器。
    docker守护进程的--dns服务配置（默认为8.8.8.8和8.8.4.4）
    宿主机上的DNS设置


####  容器配置dns ####
1)启动镜像时配置dns, docker run --dns
```shell 
# docker run --dns 172.20.11.246 --dns 114.114.114.114 -it centos bash
```

2)在docker build时配置
在/etc/docker/daemon.json文件添加dns配置
``` json
{
    "dns":[172.20.11.246]
}
```


#### docker 无法获取宿主机dns的原因及解决 ####
1)docker 无法获取宿主机的dns

2)docker run --dns指定dns也报错
```shell
# docker run --dns 172.20.11.246 172.20.8.199:5001/library/centos bash 
WARNING: IPv4 forwarding is disabled. Networking will not work.
```

解决办法：
vim /usr/lib/sysctl.d/00-system.conf
``` shell 
# 添加一行
net.ipv4.ip_forward = 1
```

然后重启网络服务
``` shell
# systemctl restart network
```
再次启动容器，解析正常了

