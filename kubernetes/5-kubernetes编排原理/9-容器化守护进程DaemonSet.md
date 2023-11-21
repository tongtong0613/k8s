##**知识点**
- DaemonSet的主要作用是可以在Kubernetes集群里运行一个Daemon Pod。
- DaemonSet和Deployment非常相似，只不过没有replicas字段。
- DaemonSet的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理Pod的情况，来决定是否创建或者删除一个节点。
- DaemonSet创建每个Pod时，会给这个Pod自动加上一个nodeAffinity，从而保证这个Pod只会在指定节点启动。同时，给这个Pod自动加上一个Toleration，从而忽略节点的unschedulable污点。
- Taint是给node设置的，Toleration是给Pod设置的。
- Deployment管理ReplicaSet对象，DaemonSet管理Pod对象，StatefulSet管理Pod对象。
- Deployment通过ReplicaSet对象管理版本，DaemonSet和StatefulSet通过ControllerRevision对象管理版本。

##**DaemonSet**
DaemonSet的主要作用是可以在Kubernetes集群里运行一个Daemon Pod。这个Pod有以下特征：
1. 这个Pod在Kubernetes集群里的没一个节点上运行。
2. 每个节点上只有一个这样的Pod实例。
3. 当有新节点加入Kubernetes集群后，该Pod会自动在新节点上被创建出来；而当旧节点被删除后，它上面的Pod也会相应地被回收。

下面是一个DaemonSet的YAML文件：
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
sepc:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v3.0.0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
这个DaemonSet使用selector管理所有携带了name: fluentd-elasticsearch标签的Pod。这些Pod模板也是用template字段定义的。

DaemonSet使用nodeAffinity在指定的节点新建Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoreDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operation: In
            values:
            - node-ituring
```
在这个Pod里，声明了一个spec.affinity字段，然后定义了一个nodeAffinity。requiredDuringSchedulingIgnoreDuringExecution意味着必须在每次调度时予以考虑；matchExpressions只允许在metadata.name是node-ituring的节点上运行。

DaemonSet Controller在创建Pod时，自动在这个Pod的API对象里加上这样一个nodeAffinity定义。

DaemonSet还会给这个Pod自动加上另外一个与调度相关的字段：tolerations。该字段意味着这个Pod会容忍某些节点的污点。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```
这个Toleration的含义是：容忍所有被标记为unschedulable污点的节点，容忍的效果是允许调度。

假如当前DaemonSet管理的是一个网络插件的Agent Pod，那么就必须在这个DaemonSet的YAML文件里给它的Pod模板加上一个能容忍`node.kubernetes.io/network-unavailable`污点的Toleration：
```yaml
...
template:
  metadata:
    name: network-plugin-agent
  spec:
    tolerations:
    - key: node.kubernetes.io/network-unavailable
      operator: Exists
      effect: NoSchedule
```

在Kubernetes项目中，当一个节点的网络插件尚未安装时，该节点就会自动被加上`node.kubernetes.io/network-unavailable`这个污点。而通过这样一个Toleration，调度器在调度这个Pod时就会忽略当前节点上的污点，从而成功将网络插件的Agent组件调度到这台机器上启动起来。

这种机制正是在部署Kubernetes集群时能够先部署Kubernetes，再部署网络插件的根本原因：因为创建Weave的YAML其实就是一个DaemonSet。