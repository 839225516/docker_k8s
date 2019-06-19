##### 将节点脱离调度，使它不再部署新的Pod
```shell
查看 node NAME
# kubectl get nodes

根据node-NAME 将node 设置为不可以使用
# kubectl cordon {node-NAME}

将node 设置为可以使用
# kubectl uncordon {node-NAME}
```

或者使用下面的方式    
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


