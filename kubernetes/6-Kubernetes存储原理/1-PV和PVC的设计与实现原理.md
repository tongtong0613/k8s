## **知识点**
- PVC描述的是Pod想要使用的持久化存储的属性，比如存储的大小、读写权限等。
- PV描述的是一个具体的Volume的属性，比如Volume类型、挂载目录、远程存储服务器地址等。
- StorageClass的作用是充当PV的模板，并且只有同属于一个StorageClass的PV和PVC才能绑定。
- StorageClass的另一个作用是指定PV的Provisioner。如果此时存储插件支持Dynamic Provisioning，Kubernetes就会自动创建PV。

## **PV与PVC**
PV描述的是持久化存储数据卷，这个API对象主要定义的是一个持久化存储在宿主机上的目录，比如一个NFS的挂载目录：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```
PVC描述的是Pod所希望使用的持久化存储的属性。比如，Volume存储的大小、可读写权限等。PVC通常由用户创建，或者以PVC模板的方式成为StatefulSet的一部分，然后由StatefulSet控制器负责创建带编号的PVC：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```
PVC要真正被容器使用，必须先与某个符合条件的PV绑定，需要满足以下条件：
1. PV与PVC的storageClassName字段必须相同。
2. PV的storage大小必须满足PVC的要求。

PV与PVC绑定之后，Pod就可以声明使用这个PVC了：
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
      - name: nfs
        mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```
Kubernetes中存在一个专门处理持久化存储的控制器`Volume Controlelr`，它维护着多个控制循环，其中`PersistVolumeController`负责绑定PV与PVC。`PersistentVolumeController`会不断查看当前每一个PVC是否已经处于`Bound`状态。如果不是，它就会遍历所有可用的PV，尝试与这个PVC进行绑定。

PV与PVC进行绑定，就是将这个PV对象的名字填在了PVC对象的`spec.volumeName`字段上。

## **持久化宿主机目录**

所谓容器的Volume，就是将一个宿主机上的目录跟容器里的目录绑定挂载在了一起。而持久化Volume，指的就是该宿主机上的目录具备持久性，即该目录里的内容不会因为容器的删除而被清理，也不会跟当前的宿主机绑定。这样，当容器重启或者在其他节点上重建后，仍能通过挂载Volume访问到这些内容。

之前使用的`hostPath`和`emptyDir`不具备持久化这个特征。大多数情况下，持久化Volume的实现需要依赖一个远程存储服务，比如远程文件存储（NFS、GlusterFS）、远程块存储（公有云提供的远程磁盘）等。

Kubernetes要做的，就是使用这些存储服务来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用。这个过程分为两阶段：

### **第一阶段：Attach**

Attach主要就是为虚拟机挂载远程磁盘。在具体Volume插件的实现接口上，在这个阶段，Kubernetes提供的可用参数是nodeName，即宿主机的名字。

Attach操作，是由`Volume Controller`负责维护的，这个控制循环叫做`AttachDetachController`。它的作用就是不断检查每一个Pod对应的PV和该Pod所在宿主机之间的挂载情况，从而决定是否需要对这个PV进行Attach或Detach操作。

`AttachDetachController`运行在Master节点。

### **第二阶段：Mount**

Mount主要做的是将磁盘设备格式化并挂载到Volume宿主机目录。在具体Volume插件的实现接口上，在这个阶段，Kubernetes提供的可用参数是dir，即Volume的宿主机目录：
```
/var/lib/kubelet/pods/<Pod ID>/volumes/kubernetes.io~<Volume 类型>/<Volume 名字>
```

Mount操作，必须发生在Pod对应的宿主机上，所以它必须是kubelet组件的一部分。这个控制循环称为`VolumeManagerReconciler`，是一个独立于kubelet主循环的Goroutine。


## **StorageClass**
Kubernetes提供了一套可以自动创建PV的机制：`Dynamic Provisioning`。该机制的工作核心是StorageClass对象。StorageClass对象的作用其实就是创建PV的模板：
```yaml
apiVersion: storage.k8s.io
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
创建这个StorageClass对象之后，只需要在PVC中声明使用这个StorageClass，就会自动创建一个PV对象，并与PVC绑定：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
sepc:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 30Gi
```

之前在StatefulSet存储状态的示例中，没有声明StorageClass。实际上，如果集群开启了名为`DefaultStorageClass`的Admission Plugin，它就会为PVC和PV自动添加一个磨人的StorageClass；否则，PVC的StorageClassName的值就是“”，这意味着它只能和StorageClassName也是“”的PV绑定。
