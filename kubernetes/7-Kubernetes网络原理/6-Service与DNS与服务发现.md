## **知识点**
- pass

## **Service**
Kubernetes之所以需要Service，一方面是因为Pod的IP不是固定不变的，另一方面是因为一组Pod实例之间总会有负载均衡的需求。

一个典型的Service：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```
这个Service只代理携带了app:hostnames标签的Pod，80端口代理Pod的9376端口。

应用的Deployment如下所示：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```

该应用的作用是每次访问9376端口时返回自己的hostname。

而被selector选中的Pod，就称为Service的Endpoints，可以使用kubectl get ep命令看到它们，如下所示：

```
$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
```

要注意的是，只有处于Running状态，且readinessProbe 查通过的Pod，才会出现在Service的Endpoints列表里。并且，当某一个Pod出现问题时，Kubernetes会自动把它从Service里摘除掉。

而此时，通过该Service的VIP地址 10.0.1.175，你就可以访问到它所代理的Pod了：

```
$ kubectl get svc hostnames
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.0.1.175   <none>        80/TCP    5s
 
$ curl 10.0.1.175:80
hostnames-0uton
 
$ curl 10.0.1.175:80
hostnames-yp2kp
 
$ curl 10.0.1.175:80
hostnames-bvc05
```

这个VIP地址是Kubernetes自动为Service分配的。而像上面这样，通过三次连续不断地访问Service的VIP地址和代理端口80，它就为我们依次返回了三个Pod的hostname。这也正印证了Service提供的是Round Robin方式的负载均衡。对于这种方式，我们称为：ClusterIP模式的Service。


## **Service工作原理**

Service是有kube-proxy组件加上iptables共同实现的。一旦Service被提交给Kubernetes，kube-proxy就会通过Service的Informer感应到一个Service对象的添加，事件的响应是在宿主机上创建一条iptables规则：

```
iptables -A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames:cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWX5X2332I4OT4T3
```

这条规则的含义是，凡是目的地址是10.0.1.175，目的端口是80的IP包，都跳转到KUBE-SVC-NWX5X2332I4OT4T3这条iptables链进行处理，KUBE-SVC-NWX5X2332I4OT4T3是一组规则的集合：

```
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
 
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
 
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR

```

这三条链指向的最终目的地，实际上就是这个Service代理的3个Pod。

```
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
 
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376
 
-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376

```

这三条链是DNAT规则，在DNAT规则之前，iptables对流入的IP包设置了一个标志 --set-mark。

DNAT规则的作用是在PREROUTING检查点之前，将流入IP包的目的地址和端口改为 --to-destination指定的新目的地址和端口。

kube-proxy通过iptables处理Service的过程，需要在宿主机创建相当多的iptables规则。在大规模项目中，会占用大量宿主机CPU资源。


## **IPVS模式**

为kube-proxy设置--proxy-mode=ipvs来开启这个功能。

IPVS工作原理和iptables模式类似，当创建了Service之后，kube-proxy会在宿主机创建一个虚拟网卡（kube-ipvs0），并为它分配VIP作为IP地址：

```
# ip addr
  ...
  73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
  link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
  inet 10.0.1.175/32  scope global kube-ipvs0
  valid_lft forever  preferred_lft forever
```

接下来，kube-proxy通过Linux的IPVS模块为这个IP地址设置三台IPVS虚拟主机，并设置这三台虚拟主机之间使用轮询模式来作为负载均衡策略：

```
# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    ->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
  TCP  10.102.128.4:80 rr
    ->  10.244.3.6:9376    Masq    1       0          0         
    ->  10.244.1.7:9376    Masq    1       0          0
    ->  10.244.2.3:9376    Masq    1       0          0
```

这三台虚拟主机的IP地址和端口正是被代理的三个Pod。

IPVS在内核中的实现也是基于Netfilter的NAT模式，因此其转发性能没有显著提升。但是，IPVS不需要造宿主机上创建iptables规则，而是把这些规则放到了内核态，从而减少了维护这些规则的代价。

需要注意的是，IPVS只负责负载均衡和代理功能，而一个完整Service流程需要的包过滤、SNAT等操作还是需要靠iptables来实现。

## **DNS**

对于ClusterIP模式的Service和Headless Service来说，他们的A记录格式都如下所示：

```
<my-svc>.<my-namespace>.svc.cluster.local
```
对于ClusterIP模式的Service，访问这条A记录，解析到的就是该Service的VIP地址；对于Headless Service来说，访问这条A记录，得到的是所有被代理的Pod的IP地址的集合。

对于ClusterIP模式的Service来说，它代理的Pod被自动分配的A记录格式是：

```
<pod-ip>.<my-namespace>.pod.cluster.local
```

对于Headless Service来说,它代理的Pod被自动分配的A记录是：

```
<pod-name>.<service-name>.<my-namespace>.svc.cluster.local
```

但是如果为Pod指定了Headless Service，并且Pod本身声明了hostname和subdomain字段，此时Pod的A记录会变成：

```
<hostname>.<subdomain>.<my-namespace>.svc.cluster.local
```


普通service是通过kubeProxy找到Pod，而kubeProxy的两种模式 iptables 和 ipvs 底层都是通过 iptables (或 ipvsadm) 规则来的；但是无头service不需要通过kubeProxy，自己直接找到Pod，既然不需要通过 kubeProxy，自然就是没有 iptables 规则。

普通Service和无头Service还有一个访问区别：要访问service，在集群内可以使用 serviceName.ns 和 clusterIp:node/宿主机IP:NodePort，但是集群外只能使用 宿主机IP:NodePort，所以普通service可以集群内和集群外都适用，无头service只能集群内使用。
