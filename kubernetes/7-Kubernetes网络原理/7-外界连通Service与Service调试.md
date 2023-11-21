## **知识点**
- pass

## **外部访问Service**

### **NodePort**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    port: 30080
    targerPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    port: 30443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```
这个YAML定义了一个NodePort类型的Service，声明了用Service的8080端口代理Pod的80端口，用Service的443端口代理Pod的443端口。此时要访问这个Service，只需要访问：

```
<集群中任一台宿主机IP>:8080
```

kube-proxy为宿主机生成了下面的iptables规则:

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-xxxxxxxxxxxx
```

接下来的流程就和CLusterIP模式一样了。但是在IP包离开宿主机发往Pod时，需要对IP包进行一次SNAT操作：

```
-A POSTROUTING -m comment --comment "SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

这条规则设置在POSTROUTING检查点，在即将离开这个主机时，对IP包进行了一次SNAT操作。MASQUERADE意思是自动读取eth0现在的ip地址然后做SNAT出去，这样就实现了很好的动态SNAT地址转换。IP包的原地址替换成这台宿主机的CNI网桥地址或者宿主机本身的IP地址（网桥不存在）。

### **LoadBalancer**

这种方式适用于公有云上的Kubernetes服务。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
```

公有云Kubernetes服务力，提供了一个CloudProvider的转接层来与公有云本身API对接。上述LoadBalancer类型Service提交后，Kubernetes会自动调用CloudProvider在公有云上创建一个负载均衡服务，并把被代理的Pod的IP地址配置给负载均衡服务作为后端。

### **ExternalName**

```yaml
apiVerrsion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
```

这个YAML文件不需要指定selector。此时，当通过DNS访问服务时，例如my-service.default.svc.cluster.local，Kubernetes返回的就是my.database.example.com。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```
指定externalIPs后，就可以通过80.11.12.10访问到被代理的Pod了。

## **Service排查错误**

当service无法通过DNS访问时，首先检查Kubernetes自己的Master节点的Service DNS是否正常：

```
# 在一个Pod里执行

$ nslookup kubernetes.default
Server:     10.0.0.10
Address 1:  10.0.0.10   kube-dns.kube-system.svc.cluester.local

Name:       kubernetes.default
Address 1:  10.0.0.1    kubernetes.default.svc.cluster.local
```

如果上面访问kubernetes.default的返回值有问题，就需要检查kube-dns运行状态和日志，否则应该检查Service的定义是否有问题。

****

如果Service无法通过ClusterIP访问到，应该首先检查这个Service是否有Endpoints：

```
$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
```

如果Pod的readniesProbe没通过，它也不会出现在Endpoints列表里。

如果Endpoints正常，就需要检查kube-proxy是否正常运行。

如果kube-proxy正常，就应该查看宿主机的iptables规则：
- KUBE-SERVICE或KUBE-NODEPORTS规则对应的Service入口链，这些规则应该与VIP和Service端口一一对应。
- KUBE-SVC-(hash)规则对应的负载均衡链，这些规则数目应该与Endpoints数目一致。
- KUBE-SEP-(hash)规则对应的DNAT链，这些规则应该与Endpoints一一对应。
- 如果是NodePort模式，还涉及POSROUTING处的SNAT链。

****

如果Pod无法通过Service访问自己，需要检查kubelet的hairpin-mode是否正确设置为hairpin-veth或者promiscuous-bridge。

