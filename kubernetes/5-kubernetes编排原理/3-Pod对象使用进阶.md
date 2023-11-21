##**知识点**
- pass
  
##**Projected Volume**
Projected Volume（投射数据卷）是一种特殊的Volume，其存在的意义不是为了存放容器里的数据，也不是用于容器和宿主机之间的数据交换，而是为容器提供预先定义好的数据。常用Projected Volume共有以下4种：
1.  Secret
2.  ConfigMap
3.  Downward API
4.  ServiceAccountToken
###**Secret**
Secret的作用是把Pod想要访问的加密数据存放到etcd种，这样就可以通过在Pod的容器中挂载Volume的方式访问这些Secret里保存的信息了。
Secret最典型的使用场景就是存放数据库的Credential信息，示例如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-crad
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: mysecret
```
这个Pod声明的容器中，挂载的Volume不是常见的emptyDir或者hostPath类型，而是projected类型。这个Volume的数据来源（sources），则是名为mysecret的Secret对象。
可以使用YAML文件创建Secret对象：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YMRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```
Secret对象要求数据必须时经过Base64转码的，以免出现明文密码，可通过以下方式进行转码：
```
echo -n 'admin' | base64
YMRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```
###**ConfigMap**
ConfigMap与Secret类似，区别是ConfigMap保存的是无需加密的、应用所需的配置信息。除此之外，与Secret用法几乎完全相同：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ui-config
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
```
###**Downward API**
Downward API的作用是让Pod里的容器能够直接获取这个Pod API对象本身的信息：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```
在这个Pod的YAML中，定义了一个容器，声明了一个projected类型的Volume。只不过这次Volume的数据来源变成了DownwardAPI，而这个Downward API Volume声明了暴露Pod的metadata.labels信息给容器。
通过这样的声明方式，当前Pod的Labels字段的值就会被Kubernetes自动挂载成为容器里的/etc/podinfo/labels文件。 

当前Downward API支持的字段如下：

1. 使用fieldRef可以声明使用：
    - metadata.name---Pod的名字
    - metadata.namespace---Pod的Namespace
    - metadata.uid---Pod的UID
    - metadata.labels[`'<KEY>'`]---指定`<KEY>`的Label值
    - metadata.annotations[`'<KEY>'`]---指定`<KEY>`Annotation值
    - metadata.labels---Pod的所有Label
    - metadata.annotations---Pod的所有Annotation

2. 使用resourceFieldRef可以声明使用：
    - 容器的CPU limit
    - 容器的CPU request
    - 容器的memory limit
    - 容器的memory request
    - 容器的ephemeral-storage limit
    - 容器的eohemeral-storage request
  
3. 通过环境变量声明使用：
    - status.PodIP---Pod的IP
    - spec.serviceAccountName---Pod的ServiceAccount名字
    - spec.nodeName---Node的名字
    - status.hostIP---Node的IP

Downward API能够获取的信息一定是Pod里的容器进程启动之前就能确定的信息，如果想要获取Pod容器运行后才会出现的信息，比如容器进程的PID，就不能使用Downward API，而应该考虑sidecar方式。

###**ServiceAccountToken**
Sevice Account对象是Kubernetes系统内置的一种服务账户，是Kubernetes进行权限分配的对象。像Service Account这样的授权信息和文件，实际上保存在它所绑定的一个特殊的Secret对象，即ServiceAccountToken。

Kubernetes集群里的每一个Pod都自动声明了一个类型是Secret、名为default-token-xxxx的Volume，然后挂载在每个容器的固定目录上：
```
$ kubectl describe pod nginx-deployment-xxxxxxxxx
Containers:
...
    Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-xxxxx (ro)
Volumes:
    default-token-xxxxx:
    Type:   Secret(a volume populated by a Secret)
    SecretName: default-token-xxxxx
    Optianal:   false
```

这个Secret类型的Volume，正是默认Sevice Account对应的Service AccountToken。所以，在每个Pod创建的时候，Kubernetes自动在它的spec.volumes部分添加了默认ServiceAccountToken的定义，然后自动给每个容器加上了对应volumeMounts字段。

一旦Pod创建完成，容器里的应用就可以直接从默认的ServiceAccountToken的挂载目录里访问授权信息和文件。这个路径是固定的：/var/run/secrets/kubernetes.io/serviceaccount。这个Secret类型的Volume的内容如下所示：
```
$ ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt  namespace token
```
此时应用程序只要直接加载这些授权文件，就可以访问并操作Kubernetes API了。如果使用的是Kubernetes官方的Client包`（k8s.io/client-go）`的话，它还可以自动加载这个目录下的文件，不需要做任何配置。

>这种把Kubernetes客户端以容器的方式在集群中运行，然后使用默认Service Account自动授权的方式，称为**InClusterConfig**。

##**容器健康检查和恢复机制**
在Kubernets中，可以为Pod里的容器定义一个健康检查“探针”（Probe）。这样kubelet就会根据Probe的返回值确定这个容器的状态，而不是直接以容器是否运行作为依据。
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
在这个Pod中定义了一个容器，它在启动之后做的第一件事就是在/tmp目录下创建了一个healthy文件，以此作为自己已经正常运行的标志。30秒后它会删除这个文件。

与此同时，定义了一个livenessProbe。它的类型是**exec**，这意味着容器启动之后会在容器中执行指定的命令：`cat /tmp/healthy`。此时如果文件存在，这条命令的返回值就是0，Pod会认为这个容器不仅已经启动，而且是健康状态。

initialDelaySeconds意味着健康检查在容器启动5秒后开始执行，periodSeconds意味着每5秒执行一次。

在30秒后查看这个Pod的状态：
```
$ kubectl get pod test-liveness-exec
NAME            READY   STATUS  RESTARTS    AGE
liveness-probe  1/1     Running 1           1m
```
Pod并没有进入Failed状态，而是保持Running状态。而RESTARTS字段变为1，这就说明异常容器已经被重启了（其实是重新创建了容器）。

这就是Kubernetes的**Pod恢复机制**，也叫restartPolicy。他是Pod的Spec部分的一个标准字段（pod.spec.restartPolicy），默认值是Always，即无论容器何时发生异常，它一定会被重新创建。

Pod的恢复过程永远发生在当前节点，一旦一个Pod与一个节点绑定，除非绑定发生了变化(pod.spec.node字段被修改),否则它永远不会离开这个节点。

如果想让Pod出现在其他可用节点上，就必须使用Deployment这样的控制器来管理Pod。

restartPolicy一共有三种：
1. Always：在任何情况下，只要容器不在运行状态，就自动重启容器。
2. OnFailure: 只在容器异常时才自动重启容器。
3. Never：从不重启容器。

restartPolicy和Pod里容器的状态以及Pod状态的对应关系：
1. 只要Pod的restartPolicy指定的策略允许重启异常的容器，那么这个Pod就会保持Running状态并重启容器，否则Pod会进入Failed状态。
2. 对于包含多个容器的Pod，只有其中所有容器都进入异常状态后，Pod才会进入Failed状态。在此之前，Pod均为Running状态，Pod的READY字段会显示正常容器的个数。

livenessProbe也可以定义为发起HTTP或者TCP请求的方式：
```yaml
...
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome
  initialDelaySeconds: 3
  periodSeconds: 3
...
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```
还有一个readinessProbe，用法与livenessProbe类似，作用却不同。readinessProbe检查结果决定了这个Pod能否通过Service的方式访问。

##**PodPreset**
首先定义一个PodPreset对象。在这个对象中，预先定义好在Pod里追加的字段：
```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  voluMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```
这个PodPreset对象的定义中，首先是一个selector，这就意味着后面这些追加的定义只会作用于selector所定义的、带有role:frontend标签的Pod对象。

先创建以上PodPreset对象，在创建下面的Pod对象：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    role: frontend
    app: website
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80
```
Pod运行起来之后查看这个Pod的API对象：
```yaml
$ kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    role: frontend
    app: website
  annotations:
    podpred.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: nginx
      voluMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```
Pod还被加上了annotation表示这个Pod对象被PodPreset修改过。

如果定义了同时作用于一个Pod对象的多个PodPreset，Kubernetes会合并(merge)多个PodPreset要做的修改，如果修改有冲突，这些冲突字段不会被修改。