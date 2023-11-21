## **知识点**
- Deployment只能处理无状态应用。
- 实际场景中，分布式应用的多个实例之间往往存在依赖关系，例如主从关系、主备关系。数据存储类应用的多个实例往往会在本地磁盘上保存数据，这种实例一旦结束重新创建出来后，实例与数据之间的对应关系已经丢失，导致应用失败。
- 这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，称为**有状态应用**。
- StatefulSet控制器的主要作用之一，就是使用Pod模板创建Pod时对它们进行编号，并且按照编号逐一完成创建工作。而当StatefulSet的控制循环发现Pod的实际状态与期望状态不一致，需要新建或删除以进行调谐时，它会严格按照这些Pod的编号的顺序逐一完成操作。
- 通过Headless Service的方式，StatefulSet为每个Pod创建了一个固定并且稳定的DNS记录，来作为它的访问入口。

## **StatefulSet**
StatefulSet将现实世界里的应用状态抽象为两种情况：
1. 拓扑状态。应用的多个实例之间不是完全对等的。这些应用实例必须按照某种顺序启动。比如应用的主节点A要先于从节点B启动。而如果删除A和B两个Pod，它们再次被创建出来时也必须严格按照这个顺序运行。并且，新创建出来的Pod必须和远来Pod的网络标识一样，这样原先的访问者才能使用相同的方法访问到新的Pod。
2. 存储状态。应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A第一次读取到的数据和隔了十分钟之后再次读取到的数据应该是同一份，哪怕在此期间Pod被重新创建过。这种情况最典型的例子是一个数据库应用的多个存储实例。
   
### **Headless Service**
Service时Kubernets中用来将一组Pod暴露给外界访问的一种机制。比如，一个Deployment有3个Pod，就可以定义一个Service。这样，用户只要能够访问这个Service，就能访问到某个具体的Pod。

Service是如何被访问的呢？
- 第一种是以Service的VIP（virtual IP，虚拟IP）方式。比如当访问10.0.23.1这个Service的IP地址时，10.0.23.1就是一个VIP，他会把请求转发到该Service所代理的某一个Pod上。
- 第二种是以Service的DNS方式。比如访问“my-svc.my-namespace.svc.cluster.local”这条DNS记录，就可以访问到名为my-svc的Service所代理的某一个Pod。

第二种Service DNS方式下，具体又可分为两种：
- 第一种处理方式是Normal Service。在这种情况下，访问“my-svc.my-namespace.svc.cluster.local”解析到的，正式my-svc这个Service的VIP，后面流程与VIP方式一致。
- 第二种处理方式是Headless Service。在这种情况下，访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是my-svc代理的某一个Pod的IP地址。区别就在于，Headless Service不需要分配VIP，而以DNS记录方式解析出被代理的Pod的IP地址。

下面是一个Headless Servicede YAML文件：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```
Headless Service仍然是一个标准的Service，只不过它的clusterIP字段设置为None，即这个Service没有一个VIP作为头。上述Headless Service代理所有携带了app：nginx标签的Pod。

当创建了上述Headless Service后，它所代理的所有Pod的IP地址都会被绑定一个如下格式的DNS记录：
```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```
这个DNS记录，正是Kubernetes为Pod分配的唯一**可解析身份**。有了这个可解析身份，只要知道Pod的名字及其对应Service的名字，就可以通过这条DNS记录访问到Pod的IP地址。

### **StatefulSet应用**
一个StatefulSet的YAML文件如下所示：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```
这个YAML文件和前面用到的nginx-deployment的唯一区别，就是多了一个serviceName=nginx字段。这个字段就是告诉StatefulSet控制器，在执行控制循环时使用nginx这个Headless Service来保证Pod可解析。

创建上述Headless Service以及StatefulSet后，就会创建出两个Pod，分别命名为web-0和web-1。StatefulSet给它管理的Pod命名时编号规则是“-”，并且编号从0开始累加。

这些Pod的创建也是严格按照编号顺序进行的，在web-0进入Running Man状态并且Condition变为Ready之前，web-1会一直处于Pending状态。

当Pod创建完之后，用nslookup命令解析Pod对应的Headless Service：
```
$ nslookup web-0.nginx
Server:     10.0.0.10
Address 1:  10.0.0.10   kube-dns.kube-system.svc.cluester.local

Name:       web-0.nginx
Address 1:  10.244.1.7

$ nslookup web-1.nginx
Server:     10.0.0.10
Address 1:  10.0.0.10   kube-dns.kube-system.svc.cluester.local

Name:       web-1.nginx
Address 1:  10.244.2.7
```
nslookup命令的输出结果显示，在访问web-0.nginx时，最后解析到的正是web-0这个Pod的IP地址。

如果删除这两个Pod，Kubernetes会按照原先编号重新创建出两个Pod。并且Kubernetes为它们分配了与原来相同的网络身份：web-0.nginx和web-1.nginx。此时再查看：
```
$ nslookup web-0.nginx
Server:     10.0.0.10
Address 1:  10.0.0.10   kube-dns.kube-system.svc.cluester.local

Name:       web-0.nginx
Address 1:  10.244.1.8

$ nslookup web-1.nginx
Server:     10.0.0.10
Address 1:  10.0.0.10   kube-dns.kube-system.svc.cluester.local

Name:       web-1.nginx
Address 1:  10.244.2.8
```
尽管DNS记录没变，但是解析到的Pod的IP地址发生了变化。这就意味着，队友状态应用的访问，必须使用DNS记录或者hostname的方式，而不能直接访问这些Pod的IP地址。
