## **知识点**
- Kubernetes网络模型
  - 所有容器都可以直接用IP地址与其他容器通信，无需使用NAT
  - 所有宿主机都可以直接使用IP地址与所有容器通信，无需使用NAT，反之亦然
  - 容器自己看到的自己的IP地址，和别人（宿主机或者容器）看到的地址完全一样

## **CNI网桥**
Kubernetes是通过一个叫作CNI的接口维护了一个单独的网桥来代替docker0，这个网桥叫作CNI网桥，在宿主机上的默认设备名称是cni0。

CNI的设计思想是：Kubernetes在启动Infra容器后，就可以直接调用CNI网络插件，为这个Infra容器的Network Namespace配置符合预期的网络栈。

要实现一个面向Kubernetes的容器网络方案，需要做两部分内容，以Flannel为例：
1. 实现这个网络方案本身，主要室编写flanneld进程里的主要逻辑。比如，创建和配置flannel.1设备，配置宿主机路由，配置ARP和FDB表里的信息。
2. 实现网络方案对应的CNI插件。这部分主要是配置Infra容器的网络栈并把它连接到CNI网桥上。

## **CNI插件工作原理**

当kubelet组件创建Pod时，首先会创建Infra容器。在这一步，dockershim会先调用Docker API创建并启动Infra容器，接着执行一个`SetUpPod`方法。这个方法用于为CNI插件准备参数，然后调用CNI插件为Infra容器配置网络。

### **准备参数**
需要的参数分为两部分：
1. 由dockershim设置的一组CNI环境变量。最重要的环境变量为CNI_COMMAND。取值为ADD或DEL。ADD意味着把容器添加到CNI网络，DEL意味着从CNI网络移除容器。
2. dockershim从CNI配置文件里加载得到的默认插件的配置信息。配置文件里有一个delegate字段，意思是CNI插件不会亲自上阵，而是会调用delegate指定的某种CNI内置插件完成任务。对于Flannel则会调用CNI Bridge插件。


### **配置网络**

首先，CNI Bridge插件在宿主机检查CNI网桥是否存在，不存在则创建，相当于在宿主机执行：
```
# 宿主机
$ ip link add cni0 type bridge
$ ip link set cni0 up
```

然后，CNI Bridge插件会通过Infra容器的Network Namespace文件进入这个Network Namespace中，创建一堆Veth Pair设备。接着把这个Veth Pair设备的一端移动到宿主机上，相当于在容器中执行：
```
# 容器

# 创建一堆Veth Pair设备
$ ip link add eth0 type veth peer nama vethb4963f3

# 启动eth0
$ ip link set eth0 up

# 将另一端放到宿主机
$ ip link set vethb4963f3 netns $HOST_NS

# 通过Host Namespace启动宿主机上的vethb4963f3设备
$ ip netns exec $HOST_NS ip link set vethb4963f3 up
```

接下来，CNI Bridge插件把vethb4963f3设备连接到CNI网桥上，相当于执行：
```
# 宿主机
$ ip link set vethb4963f3 master cni0
```

vethb4963f3设备连接到cni0网桥后，CNI Bridge插件会为其设置Hairpin Mode。这是因为默认情况下，网桥设备不允许一个数据包从一个端口进来后，在从该端口发出。设置发夹模式后，取消这个限制。

这个特定主要用于容器需要通过NAT方式自己访问自己的场景。

接下来， CNI Bridge插件调用CNI IPAM插件，从ipam.subnet字段规定的网段里为容器分配一个可用IP地址。然后， CNI Bridge插件会把这个IP地址添加到容器的eth0网卡上同时为容器设置默认路由，相当于执行：

```
# 容器
$ ip addr add 10.244.0.2/24 dev eth0
$ ip route add default via 10.244.0.1 dev eth0
```

最后，CNI Bridge插件为cni0网桥添加IP地址：

```
# 宿主机
$ ip addr add 10.244.0.1/24 dev cni0
```

执行完上述操作后，CNI 插件会把容器的IP地址等信息返回给dockershim，然后被kubelet添加到Pod的Status字段。
