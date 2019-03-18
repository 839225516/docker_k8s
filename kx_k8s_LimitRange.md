#### k8s使用LimitRange对namespace级做资源限制
对namespace: product设置CPU和内存资源限制

    设置POD创建时要求Node节点必须有空闲资源：   500mCPU 和 1Gi内存
    设置POD最大能使用的CPU和内存资源:          2CPU 和 4Gi内存
    设置Container创建时默认的最小需求资源:      500mCPU 和 2Gi内存      



product-limitrange.yaml
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  namespace: product
  name: cpu-memory-limits
spec:
  limits:
  - max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 500m
      memory: 1Gi
    type: Pod
  - default:
      cpu: "2"
      memory: 4Gi
    defaultRequest:
      cpu: 500m
      memory: 1Gi
    max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 500m
      memory: 512Mi
    type: Container
```
```shell
kubectl create -f product-limitrange.yaml

kubectl describe limitrange -n product 

Name:       mylimits
Namespace:  product
Type        Resource  Min   Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Pod         cpu       500m   2    -                -              -
Pod         memory    1Gi    4Gi  -                -              -
Container   cpu       500m   2    500m             2              -
Container   memory    512Mi  4Gi  1Gi              4Gi            -
```

pod资源限制：

    创建阶段： 必须满足有 1Gi内存 和 500毫核CPU 的硬件资源
    使用阶段： pod允许使用的 最大内存为4Gi 和 最大CPU为2核


container资源限制：

    default:          配置resourceQuota时，创建container的 默认limit上限
    defaultRequest:   配置resourceQuota时，创建container的 默认request上限

    创建阶段:          必须满足每个容器有 500mCPU 和 512Mi内存 的硬件资源
    使用阶段：         container允许使用的 最大内存为4Gi 和 2核CPU
    其中： min <= defaultRequest <= default <= max 


##### 测试用例
1. 在namespace  product下创建一个pod，pod不指定任何资源信息,此时 limit取default的值 和 request取defaultRequest 
default-limits-demo.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-limits-demo
spec:
  containers:
  - name: default-limits-demo
    image: envoyproxy/envoy
```
```shell
#查看pod使用的CPU和内存（可以看出POD并没有独占request要求的资源，实际使用CPU和内存小于request,但它要求创建时，node在资源必须大于request）
kubectl top pod -n product
NAME                  CPU(cores)   MEMORY(bytes)   
default-limits-demo   6m           21Mi

#查看pod的资源限制
kubectl get pod default-limits-demo -o yaml -n product 

spec:
  containers:
  - image: envoyproxy/envoy
    imagePullPolicy: Always
    name: default-limits-demo
    resources:
      limits:
        cpu: "2"
        memory: 4Gi
      requests:
        cpu: 500m
        memory: 1Gi
```

2. 在namespace  product下创建一个pod，pod指定container只设置limit（只设置使用上限）,没有设置request,那么最终创建出的container,其request=limit。而不是default request
limits-demo.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limits-demo
spec:
  containers:
  - name: limits-demo
    image: envoyproxy/envoy
    resources:
      limits:
        cpu: "2"
        memory: 4Gi
```

```shell
kubectl create -f limits-demo.yaml -n product 

#查看pod的资源使用情况
kubectl top pod -n product
NAME                  CPU(cores)   MEMORY(bytes)   
limits-demo           6m           16Mi 

#查看pod的资源限制
kubectl get pod limits-demo -o yaml -n product 

spec:
  containers:
  - image: envoyproxy/envoy
    imagePullPolicy: Always
    name: limits-demo
    resources:
      limits:
        cpu: "2"
        memory: 4Gi
      requests:
        cpu: "2"
        memory: 4Gi
```

3. 在namespace  product下创建一个pod，pod指定container只设置requests时,limits则取default  
limists-min-demo.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limits-min-demo
spec:
  containers:
  - name: limits-demo
    image: envoyproxy/envoy
    resources:
      requests:
        cpu: "1"
        memory: 2000Mi
```

```shell
kubectl create -f limists-min-demo.yaml -n product 

kubectl get pod limits-min-demo -o yaml -n product 
spec:
    resources:
      limits:
        cpu: "2"
        memory: 4Gi
      requests:
        cpu: "1"
        memory: 2000Mi
```

4. 在namespace product下创建一个pod，pod指定container设置的limit 大于container 的 max时，会报错创建失败  
limits-max-demo.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limits-max-demo
spec:
  containers:
  - name: limits-demo
    image: envoyproxy/envoy
    resources:
      limits:
        cpu: "1"
        memory: 5Gi
```

```shell
kubectl create -f limits-demo.yaml -n product 

Error from server (Forbidden): error when creating "limits-max-demo.yaml": pods "limits-max-demo" is forbidden: [maximum memory usage per Pod is 4Gi, but limit is 5368709120., maximum memory usage per Container is 4Gi, but limit is 5Gi.]
```

5. 在namespace product下创建一个pod,指定container的requests和limits,但设置的limist 小于limitRange的min  
limits-set-demo.yaml  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limits-set-demo
spec:
  containers:
  - name: limits-demo
    image: envoyproxy/envoy
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
      limits:
        cpu: 500m
        memory: 2500Mi
```

```shell
kubectl create -f limits-set-demo.yaml -n product 

Error from server (Forbidden): error when creating "limits-set-demo.yaml": pods "limits-set-demo" is forbidden: [minimum cpu usage per Pod is 500m, but request is 100m., minimum memory usage per Pod is 1Gi, but request is 104857600., minimum cpu usage per Container is 500m, but request is 100m., minimum memory usage per Container is 512Mi, but request is 100Mi.]
```

**container的request和limits 必须是在[min,max]区间**

#### 总结
pod的container在不设置request和limits时，container取默认的defaultRequest和default    
pod的container在设置request和limits时，request和limit的取值必须在区间[min,max]内    
pod的container在只设置requests时,limits取默认的default   
pod的container在只设置limits时，requests=limists  






