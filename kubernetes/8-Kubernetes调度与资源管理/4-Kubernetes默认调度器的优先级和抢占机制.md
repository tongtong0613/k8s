##**知识点**
- 优先级和抢占机制解决的是Pod调度失败时该怎么办的问题

##**优先级**

调度器里维护一个调度队列，高优先级的Pod会比低优先级的Pod提前出队。

首先声明一个PriorityClass的定义：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: ""
```

优先级的值不超过1000000000，值越大优先级越高。globalDefault如果为true，就意味着这个值会成为系统默认值；如果为false，意味着只有声明使用这个PriorityClass的Pod拥有这个优先级，其他未声明的Pod优先级为0。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

##**抢占**

当一个高优先级Pod调度失败时，调度器的抢占能力会被触发。此时，调度器会试图从当前集群里寻找一个节点：当该节点上一个或多个低优先级Pod被删除后，待调度的高优先级Pod会被调度到该节点上。

抢占过程发生时，抢占者不会立即被调度到被抢占节点。而是只会将抢占者的spec.nominatedNodeName设置为被抢占节点的名字，然后抢占者会重新进入下一个调度周期。

调度队列里实现了两个队列：
- activeQ。activeQ队列里的Pod，都是下一个调度周期需要调度的对象。之前提到的入队出队指的都是从activeQ队列。
- unschedulableQ。专门存放调度失败的Pod。当这里面的Pod更新后，调度器会把Pod移动到activeQ中。

高优先级Pod调度失败后，会进入unschedulableQ队列，这次失败事件会触发调度器抢占流程：
1. 调度器检查事件失败原因，以确认能否为抢占者找到一个新节点。
2. 如果确定可以抢占，调度器会复制一份缓存的节点信息，使用这些信息模拟抢占流程，确定最佳抢占结果。
3. 得到抢占结果后，调度器开始真正的抢占操作：
    - 调度器检查牺牲者列表，清理这些Pod携带的nominatedNodeName字段
    - 调度器把抢占者的nominatedNodeName字段设置为被抢占节点的名字，这一步会触发重新做人流程，抢占者被移到activeQ队列
    - 调度器遍历牺牲者列表，向API Server发送请求逐一删除牺牲者


如果在对某一对Pod和节点执行Predicates算法时，发现待检查节点是一个即将被抢占的节点，即调度队列中存在nominatedNodeName字段值是该节点名字的Pod，调度器会对该节点执行两次预选算法:
- 第一遍，调度器假设上述潜在的抢占者已经在该节点运行，执行预选算法
- 第二遍，调度器不考虑潜在的抢占者，正常执行预选算法
