##**知识点**
- CSI的设计思想是把插件的职责从两阶段处理扩展成了Provision、Attach和Mount三个阶段。
- Provison等价于创建磁盘，Attach等价于挂载磁盘到虚拟机，Mount等价于格式化该磁盘并挂在到Volume的宿主机目录。
- AttachDetachController需要进行Attach操作时，会执行到pkg/volume/csi目录，创建一个VolumeAttachment对象，从而触发External Attacher调用CSI Controller服务的ControllerPublishVolume方法。
- VolumeManagerReconciler需要进行Mount操作时，会执行到pkg/volume/csi目录，直接向CSI Node服务发起调用NodePublishVolume方法。

##**CSI插件**

![插件存储](C:/Users/root/Desktop/kubernetes/images/插件存储.png)


这套存储插件体系拥有三个独立的外部组件：`Driver Registrar`、`External Provisioner`和`External Attacher`。

- `Driver Registrar`组件负责将插件注册到kubelet中。具体实现是通过请求CSI插件的`Identity`服务来获取插件信息。
- `External Provisioner`组件负责`Provision`阶段。具体实现上，External Provisioner监听（WATCH）API Server里的PVC对象。当一个PVC被创建时，他就会调用`CSI Controller`的`CreateVolume`方法，创建相应的PV。
- `External Attache`r组件负责`Attach`阶段。具体实现上，External Attacher监听API Server的`VolumeAttachment`对象的变化。一旦出现VolumeAttachment对象，它就会调用`CSI Controller`的`ControllerPublish`方法，完成Attach阶段。

Volume的`Mount`阶段不属于外部组件的职责。当kubelet的`VolumeManagerReconciler`控制循环检查到需要执行Mount操作时，会通过pkg/volume/csi包直接调用`CSI Node`服务完成Mount阶段。

实际使用中，会把三个外部组件作为sidecar容器和CSI插件放在同一个Pod中。一个CSI插件会以`gRPC`的方式对外提供三个服务：`CSI Identity`、`CSI Controller`和`CSI Node`。

###**CSI Identity**
CSI Identity负责对外暴露这个插件本身的信息：
```go
service Identity {
    rpc GetPluginInfo(GetPluginInfoRequest)
        returns (GetPluginInfoResponse) {}
    rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
        returns (GetPluginCapabilitiesResponse) {}
    rpc Probe(ProbeRequest)
        returns (ProbeResponse) {}
}
```

###**CSI Controller**
CSI Controller 服务定义的是对CSI Volume的管理接口，比如创建和删除CSI Volume、对CSI Volume进行Attach/Detach（CSI中称为Publish/Unpublish）,以及对CSI Volume进行快照：

```go
service Controller {
    rpc CreateVolume(CreateVolumeRequest)
        returns (CreateVolumeResponse) {}

    rpc DeleteVolume(DeleteVolumeRequest)
        returns (DeleteVolumeResponse) {}

    rpc ControllerPublishVolume(ControllerPublishVolumeRequest)
        returns (ControllerPublishVolumeResponse) {}

    rpc ControllerUnpublishVolum(ControllerUnpublishVolumeRequest)
        returns (ControllerUnpublishVolumeResponse) {}

    rpc CreateSnapshot(CreateSnapshotRequest)
        returns (CreateSnapshotResponse) {}

    rpc DeleteSnapshot(DeleteSnapshotRequest)
        returns (DeleteSnapshotResponse) {}
}
```

CSI Controller服务里定义的操作有个共同点：它们都无需在宿主机进行，而是属于Kubernetes里Volume Controller的逻辑，即属于Master节点的一部分。

CSI COntroller服务的实际调用者并不是Kubernetes（通过pkg/volume/csi发起请求），而是External Provisioner和External Attacher。这两个外部组件分别监听PVC和VolumeAttachment对象来跟Kubernetes进行协作。

###**CSI Node**
CSI Node服务定义了CSI Volume需要在宿主机进行的操作：
```go
service Node {
    rpc NodeStageVolume(NodeStageVolumeRequest)
        returns (NodeStageVolumeResponse) {}

    rpc NodeUnstageVolume(NodeUnstageVolumeRequest)
        returns (NodeUnstageVolumeResponse) {}

    rpc NodePublishVolume(NodePublishVolumeRequest)
        returns (NodePublishVolumeResponse) {}
    
    rpc NodeUnpublishVolume(NodeUnpublishVolumeRequest)
        returns (NodeUnpublishVolumeResponse) {}
    
    rpc NodeGetVolumeStats(NodeGetVolumeStatsRequest)
        returns (NodeGetVolumeStatsResponse) {}

    rpc NodeGetIndo(NodeGetIndoRequest)
        returns (NodeGetIndoResponse) {}
}
```

Mount阶段在CSI Node的接口是由NodeStageVolume和NodePublishVolume这两个接口共同实现的。

##**整体流程**

###**第一步，用户创建一个PVC**
1.  创建一个PVC后，`External Provisioner`容器就会监听到PVC被创建，调用**同一个Pod**的`CSI Controller`服务的`CreateVolume`方法，创建一个PV。
2.  **Master节点**上运行的`Volume Controller`就会通过`PersistentVolumeController`控制循环发现这对PVC-PV，并且由于其StorageClass是同一个，会将其绑定。
   
###**第二步，用户创建一个Pod并声明使用这个PVC**

假设Pod被调度到宿主机A上。

1.  **Master节点**上运行的`Volume Controller`通过`AttachDetachController`控制循环发现，上述PVC对应的Volume需要Attach到宿主机A上，就会创建一个`VolumeAttachment`对象，该对象携带了宿主机A和待处理的Volume的名字。
2.  此时，`External Attacher`容器就会监听到`VolumeAttachment`对象。它就会使用这个对象里的宿主机和Volume名字，在调用**同一个Pod**的`CSI Controller`服务的`ControllerPublishVolume`方法，完成Attach阶段。
3.  **宿主机A上运行的kubelet**的`VolumeManagerReconciler`控制循环，发现当前宿主机上有一个Volume对应的存储设备被Attach到了某个设备目录下。于是它就会调用**同一台宿主机**上CSI插件的`CSI Node`服务的`NodeStageVolume`和`NodePublishVolume`方法，完成Mount阶段。