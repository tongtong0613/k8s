##**知识点**
- 响应命令式请求(kubectl replace)执行过程是使用新的YAML文件中的API对象替换原有的API对象，一次只能处理一个写请求，否则可能发生冲突。
- 声明式请求(kubectl apply)是执行力对原有API对象的PATCH操作，一次能处理多个写操作，并且具备Merge能力。

##**Istio项目中的声明式API**

![Istio项目架构](C:/Users/root/Desktop/kubernetes/images/Istio.png)

从图中看出Istio项目最根本的组件是在每一个应用Pod里运行的Envoy容器。Istio项目以sidecar容器的方式，在每个Pod里运行这个代理服务。由于Pod里所有容器共享一个Network Namespace，所以Envoy容器能够通过配置Pod里的iptables规则来接管整个Pod的进出流量。Istio控制层里的Pivot组件就能够通过调用每个Envoy容器的API，来对这个Envoy代理进行配置，从而实现微服务治理。

Kubernetes项目中，当一个Pod或者任何一个API对象被提交到API Server后，有一些初始化性质的工作需要在它们被Kubernetes正式处理之前完成，比如给Pod加上某些标签。

初始化操作的实现，借助的是`Admission`功能，Kubernetes还提供了一种热插拔式的Admission机制，称为`Dynamic Admission Control`，也称`Initializer`。

现在有如下一个应用Pod:
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: myapp-pod
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello && sleep 300']
```

Istio项目要做的就是在这个Pod YAML提交给Kubernetes之后，在它对应的API对象里自动加上Envoy容器的配置:
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: myapp-pod
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello && sleep 300']
  - name: envoy
    image: lyft/envoy:xxxxxxxxxxxxxxxxxxxxxxxxxxx
    command: ["/usr/local/bin/envoy"]
    .**..**
```

Istio通过编写一个自动用来为Pod注入Envoy容器的Initializer完成以上操作。首先，Istio会将这个Envoy容器本身的定义以ConfigMap的方式保存在Kubernetes中：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:xxxxxxxxxxxxxxxxxxxxxxxxxxx
        command: ["/usr/local/bin/envoy"]
        args:
          - "--concurrency 4"
          - "--config-path /etc/envoy/envoy.json"
          - "--mode serve"
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
```

这个ConfigMap的data部分正是一个Pod对象的一部分定义。Initializer要做的就是把这部分Envoy相关字段自动添加到用户提交的Pod API中。Kubernetes必须使用类似于git merge这样的操作，才能将两部分内容合并在一起。所以，在Initializer更新Pod对象时，必须使用PATCH API来完成，这种PATCH API正是声明式API最主要的能力。

接下来，Istio将一个编写好的Initializer作为一个Pod部署在KUbernetes中：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envoy-initializer
  labels:
    app: envoy-initializer
spec:
  containers:
    - name: envoy-initializer
      image: envoy-initializer:0.0.1
      imagePullPolicy: Always
```

envoy-initializer:0.0.1镜像就是一个事先编写好的自定义控制器。Kubernetes控制器其实就是一个死循环：不断比较实际状态与期望状态。Initializer控制器的实际状态就是用户新创建的Pod，而期望状态是在这个Pod里添加了Envoy容器的定义：
```go
for {
    // 获取新创建的Pod
    pod := client.GetLatestPod()
    // 检查是否已经初始化过
    if !isInitialized(pod) {
        // 没有就初始化
        doSomething(pod)
    }
}
```

Istio想要往这个Pod里合并的字段，正是之前保存在ConfigMap里的数据（data字段的值）。所以Initializer控制器的工作逻辑如下：
```go
func doSomething(pod) {
    // 获取ConfigMap
    cm := client.Get(ConfigMap, "envoy-initializer")

    // 创建空的Pod对象
    newPod := Pod{}
    newPod.Spec.Containers = cm.Containers
    newPod.Spec.Volumes = cm.Volumes

    //生成patch数据
    patchBytes := strategicpatch.CreateTwoWayMergePatch(pod, newPod)

    //发起PATCH请求，修改这个Pod对象
    client.Patch(pod.Name, patchBytes)
}
```

Kubernetes还允许通过配置来指定要对什么样的资源进行Initialize操作：
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: envoy-config
initializers:
  - name: envoy.initializer.kubernetes.io
    rules:
      - apiGroups:
          - "" // 为空代表core API group
        apiVersions：
          - v1
        resources：
          - pods
```
这个配置意味着Kubernetes要对所有Pod进行Initialize操作，制定了操作的Initializer是envoy-initializer。一旦这个InitializerConfiguration被创建，Kubernetes就会把这个Initializer的名字加在所有新创建Pod的Metadata上：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
  initializers:
    pending:
      - name: envoy.initializer.kubernetes.io
```
这个Metadata正是接下来Initializer控制器判断这个Pod是否执行过自己所负责的初始化操作的依据。当Initializer完成操作后，需要清除这个标志。

除了使用上面的配置方法，还可以在具体Pod的Annotation里添加一个如下所示的字段，从而声明要使用某个Initializer：
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
   "initializer.kubernetes.io/envoy": "true"
   ...
```

这个机制得以实现，正是借住了Kubernetes能够对API对象进行在线更新的能力，这也正是Kubernetes声明式API的独特之处：
- 首先，声明式API指的就是只需要提交一个定义好的API对象来声明期望状态。
- 其次，声明式API允许有多个API写端，以PATCH的方式对API对象进行更改，无需关心本地原始YAML文件的内容。
- Kubernetes中不止用户会修改API对象，Kubernetes本身也会修改，所以必须能够处理冲突，Kubernetes中叫做Server Side Apply。
- 最后，有了上述能力，Kubernetes才可以基于对API对象的增删改查，在完全无需外界干预的情况下，完成对实际状态和期望状态的调谐。