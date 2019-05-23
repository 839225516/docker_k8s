##### 将节点脱离调度，使它不再部署新的Pod
unschedul_node.yaml
```yaml
apiVersion: V1
kind: Node
metadata:
  name: kube-node1
  labels:
    kubernetes.io/hostname: kube-node1
spec:
  unschedulable: true
```
kubectl replace -f unschedul_node.yaml   

对后续创建的Pod,系统将不会再向该Node进行调度   
另一种是直接用 kubectl patch命令完成   
kubectl patch node kube-node1 -p '{"spec":{"unschedulable" : true} }'


