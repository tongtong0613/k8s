## **知识点**
- pass

## **资源模型**

Pod是Kubernetes中最小调度单位，所有跟调度和资源管理相关属性都是Pod的字段。最重要的就是CPU和内存：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

Kubernetes中，CPU资源称为可压缩资源，当它不足时，Pod只会饥饿不会退出。
内存称为不可压缩资源，当它不足时，Pod就会因为OOM被内核结束。

Pod可以由多个Container组成，Pod整体的资源配置需要累加得到。

requests和limits区别如下：调度的时候，kube-scheduler会按照requests的值进行计算；在真正设置Cgroups限制时，kubelet会按照limits值进行设置。

用户在提交Pod时，可以声明一个较小的requests值供调度器使用，而Kubernetes真正给容器Cgroups设置的limits较大。

## **QOS模型**

Kubernetes中，不同的requests和limits值的设定，会将这个Pod划分到不同的QOS级别。

- 当Pod内每一个Container都同时设置了requests和limits，并且二者值相等时，Pod属于**Guaranteed级别**，即它的qoSClass字段被Kubernetes自动设置为Guaranteed。当Pod只设置limits但没有设置requests，也属于Guaranteed。
- 当不满足Guaranteed条件，但至少有一个Container设置了requests，这个Pod被划分到**Burstable级别**。
- 当Pod既没有设置requests也没有设置limits，它属于**BestEffort级别**。

Qos划分主要应用场景，是在宿主机资源紧张时，kubelet对Pod进行Eviction（资源回收）时用到的。Eviction分为soft和hard两种模式。hard模式在资源不足时立即回收，soft模式会在资源不足时，资源不足时间超过优雅时间才会回收。

Eviction发生时，回收Pod按照以下顺序：
- 首先回收BestEffort类别的Pod
- 其次回收Burstable类别，并且发生饥饿的资源使用量已经超出requests的Pod
- 最后是Guaranteed类别，并且Kubernetes保证只有当Guaranteed类别的Pod资源使用量超过其limits限制，或者宿主机本身处于Memory Pressure状态时，Guaranteed类别的Pod才会被选中进行Eviction操作
- 相同Qos类别的Pod，Kubernetes还会根据Pod的优先级进一步排序选择

如果想要设置使用cpuset把容器绑定到某个cpu核上，而不是使用cpushare共享CPU计算能力，在Kubernetes中只需要做到：
1. 保证Pod是Guaranteed类别
2. 将Pod的CPU资源的requests和limits设置为相等的整数值
