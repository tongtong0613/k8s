## 前言

如果想要将nginx部署到Kubernetes集群，可以在终端中输入以下命令：

> $ kubectl run --image=nginx --replicas=3

几秒后，三个nginx Pod会分布在工作节点上。接下来看一下从client到kubelet完整的生命周期。

## Kubectl

### 1. 验证和生成器

当执行上面命令后，`kubectl`首先会执行一些客户端验证操作，以确保不合法的请求（例如，创建不支持的资源或使用格式错误的镜像名称）将会快速失败，也不会发送给`kube-apiserver`。通过减少不必要的负载来提高系统性能。

验证通过后，kubectl开始将发送给kube-apiserver的HTTP请求封装。`kube-apiserver`与`etcd`进行通信，所有尝试访问或更改Kubernetes系统状态的请求都会通过kube-apiserver进行，kubectl也不例外。kubectl使用生成器来构造HTTP请求。生成器是一个用来处理序列化的抽象概念。

通过`kubectl run`不仅可以运行`Deployment`，还可以通过指定参数`--generator`来部署其他多种资源类型。如果没有指定`--generator`参数的值，kubectl会自动判断资源的类型。

例如，带有参数`--restart-policy=Always`的资源将被部署为Deployment，带有参数`--restart-policy=Never`的资源将被部署为Pod。同时kubectl也会检查是否需要触发其他操作，例如记录命令（--record）。

在kubectl判断出要创建一个Deployment对象后，它将使用`DeploymentAppsV1`生成器从提供的参数中生成一个`运行时对象`。


### 2. API版本协商与API组

Kubernetes支持多个版本，每个版本都在不同的路径下，例如`apps/v1`或者`apis/extrensions/v1beta1`，不同的API版本表明不同的稳定性和支持级别。

API组旨在对类似资源进行分类，以便使Kubernetes API更容易扩展。API组名是在REST路径或者序列化对象的`apiVersion`字段指定的。例如，Deployment的API组名是`apps`，最新的版本是`v1`，这就是为什么需要在Deployment manifests顶部输入`apiVersion： apps/v1`。

kubectl在生成运行时对象后，开始为它找到适合的API组和API版本，然后组装成一个版本化客户端，该客户端知道各种REST语义。这个阶段称为`版本协商`，kubectl会扫描`Remote API`上的`/apis`路径来检索所有可能的API组。由于kube-apiserver在`/apis`上公开了OpenAPI格式的规范文档，因此客户端很容易找到合适的API。为了提高性能，kubectl将OpenAPI规范缓存到了`~/.kube/cache/discovery`目录。

最后一步是发送HTTP请求。一旦请求发送之后获得成功的响应，kubectl会根据所需的输出格式打印出success message。


### 3. 客户端身份认证

在发送HTTP请求之前还要进行客户端身份认证。用户凭证保存在`kubeconfig`文件中，kubectl通过以下顺序来找到kubeconfig文件：

- 如果提供了`--kubeconfig`参数，kubectl就是用参数指定的kubeconfig文件。
- 如果没有提供`--kubeconfig`参数，但设置了环境变量`$KUBECONFIG`，则使用环境变量提供的kubeconfig文件。
- 如果既没有提供`--kubeconfig`参数，也没有设置环境变量 `$KUBECONFIG`，kubectl就是用默认的kubeconfig文件`$HOME/.kube/config`。

解析完kubeconfig文件后，kubectl会确定当前要使用的上下文、当前指向的群集以及与当前用户关联的任何认证信息。如果用户提供了额外的参数（例如--username），则优先使用这些参数覆盖kubeconfig文件的值。一旦拿到这些信息后，kubectl就把这些信息填充到将要发送的HTTP请求头中：

- x509证书使用`tls.TLSConfig`发送（包括CA证书）
- `bearer tokens`在HTTP请求头`Authorization`中发送
- 用户名和密码通过HTTP基本认证发送
- `OpenID`认证过程是由用户事先手动处理的，产生一个像bearer token一样被发送的token

## kube-apiserver

### 1. 认证

HTTP请求会发送到`kube-apiserver`，kube-apiserver是客户端和系统组件用来保存和检索集群状态的主要接口。为了执行相应的功能，kube-apiserver需要能够验证请求者是合法的，这个过程被称为认证。

当kube-apiserver第一次启动时，他会查看用户提供的所有CLI参数，并组合成一个合适的令牌列表。例如，如果提供了`--client-ca-file`参数，则会将x509客户端证书认证添加到令牌列表中，如果提供了`--token-auth-file`参数，则会将bearer token添加到令牌列表中。

每次收到请求时，apiserver都会通过令牌链进行认证，知道某个认证成功为止：

- x509处理程序将验证HTTP请求是否是由CA根证书签名的TLS秘钥进行编码的
- bearer token处理程序将验证--token-auth-file参数提供的token文件是否存在
- 基本认证处理程序将 确保HTTP请求的基本认证凭证与本地状态相匹配

如果认证失败，则请求失败并返回相应错误信息；如果认证成功，则将请求中的`Authorization`请求头删除，并将用户信息添加到上下文`context`中，给后续的授权和准入控制器提供了访问之前建立的用户身份的能力。

### 2. 授权

现在虽然已经证明HTTP请求是合法的，但是没有确保有权执行操作，身份和权限不同。为了后续操作，kube-apiserver还需要对用户进行授权。

kube-apiserver处理授权的方式与处理身份验证的方式相似：通过kube-apiserver的启动参数`--authorization_mode`参数设置。它将组合一系列授权者，这些授权者将针对每个传入的请求进行授权。如果所有授权者都拒绝该请求，则该请求被禁止响应并不再继续响应；如果某个授权者批准了该请求，则请求继续。

kube-apiserver支持以下几种授权方法：

- `webhook`：它与集群外的HTTP(S)服务交互
- `ABAC`：它执行静态文件中定义的策略
- `RBAC`：它使用`rbac.authorization.k8s.io`API Group实现授权决策，允许管理员通过Kubernetes API动态配置策略
- `Node`：它确保kubelet只能访问自己节点上的资源

### 3. 准入控制

从kube-apiserver的角度看，它已经验证了身份并赋予了相应的权限允许继续，但对Kubernetes而言，其他组件对于应不应该允许发生的事情还有意见。所以这个请求还要通过`Admission Controller`所控制的一个`准入控制链`的层层考验，官方标准的关卡有近十个之多，而且还支持自定义扩展。

虽然授权的重点是回答用户是否有权限，但准入控制器会拦截请求以确保它符合集群的更广泛的期望和规则。它们是资源对象保存到`etcd`之前的最后一个壁垒，封装了一系列额外的检查以确保操作不会产生意外或负面结果。不同于授权和认证只关心用户的请求和操作，准入控制还处理请求的内容，并且只对创建、更新、删除或连接(如代理)有效，对读操作无效。

> 准入控制器的工作方式与授权者和验证者的方式类似，但有一点区别：与验证链和授权链不同，如果某个准入控制器检查不通过，则整个链会中断，整个请求会立即被拒绝并返回一个错误给终端用户。

准入控制器设计的终点在于可扩展性，某个控制器都作为一个插件存储在`plugin/pkg/admission`目录中，并且与某一个几口相匹配，最后被编译到kube-apiserver二进制文件中。

准入控制器通常分为资源管理、安全、默认和引用一致性。以下是准入控制器示例：

- `InitialResources`: 基于过去的使用情况为容器资源设置默认限制
- `ResourceQuota`：不仅能限制某个Namespace中创建资源的数量，还能限制某个Namespace中被Pod所请求的资源总量。该准入控制器和资源对象`ResourceQuota`一起实现了资源配额管理
- `LimitRanger`：针对Namespace资源的每个个体(Pod与Container等)的资源配额。该插件和资源对象`LimitRange`一起实现资源管理配额
- `SecurityContextDeny`:该插件禁止创建设置了Security Context的Pod

## etcd

目前为止，Kubernetes已经对客户端的调用请求进行了全面的审查，并且已经验证通过，运行它进入下一环节。下一步kube-apiserver将对HTTP请求进行反序列化，然后利用得到的结果构建运行时对象，并保存到`etcd`中，下面将该过程分解一下。

在客户端发送调用请求之前，kube-apiserver就产生了一系列非常复杂的流程。从kube-apiserver首次运行二进制文件开始：

1. 当运行kube-apiserver二进制文件时，它会创建一个`允许apiserver聚合的服务链`，这是一种对Kubernetes API进行扩展的方式。
2. 同时会创建一个`generic apiserver`作为默认的apiserver。
3. 然后利用`生成的OpenAPI规范`来填充apiserver的配置。
4. 然后kube-apiserver遍历数据结构中指定的所有API组，并将每一个API组作为通用的存储抽象保存到etcd中。当你访问或者变更资源状态时，kube-apiserver就会调用这些API组。
5. 每个API组都会遍历它的所有组版本，并且将每个HTTP路由映射到`REST路径`中。
6. 当请求的METHOD是`POST`时，kube-apiserver就会把请求交给`资源创建处理器`。

现在kube-apiserver已经知道了所有的路由及其对应的REST路径，以便在请求匹配时知道调用哪些处理器和键值存储。现在假设客户端的HTTP请求已经被kube-apiserver收到：

1. 如果处理链可以将请求与已经注册的路由进行匹配，就会将该请求交给注册到该路由的`专用处理器`来处理；如果没有任何一个路由可以匹配该请求，就会将请求转交给`基于路径的处理器`来处理(比如当调用/apis时)；如果没有任何一个基于路径的处理器注册到该路径，请求会交给`not found处理器`，最后返回`404`。
2. 我们有一个`createhandler`处理器，它的作用是解码HTTP请求并进行基本的验证，例如确保请求提供的json与API资源版本相匹配。
3. 接下来进入审计和准入控制阶段。
4. 然后资源会通过`storage provider`保存到`etcd`中。默认情况下etcd的键的格式为`<namespace>/<name>`，也可自定义。
5. 资源创建过程中出现的任何错误都会被捕获，最后`storage provider`会执行`get`调用来确认该资源是否被成功创建。如果需要额外的清理工作，就会调用后期创建的处理器和装饰器。
6. 最后构造HTTP响应返回给客户端。

目前为止，创建的Deployment资源已经保存到etcd中，但apiserver仍然看不到它。

## 初始化

在一个资源对象被持久化到数据存储后，apiserver还无法完全看到或调度它，在此之前还要执行一系列的`Initializers`。Initializers是一种与资源类型相关联的控制器，它会在资源对外可见之前执行某些逻辑。如果某个资源类型没有Initializers，就会跳过此初始化步骤，立即使资源对外可见。

Initializers是一个强大的功能，因为它允许执行通用引导操作，例如：

- 将sidecar容器注入到暴露80端口的Pod中，或加上特定的`annotation`
- 将保存着测试证书的`volume`注入到特定命名空间的所有Pod中
- 如果`Secret`中的密码小于20字符，就组织其创建

`InitializerConfiguration`资源对象允许声明某些资源类型应该运行哪些Initializers。如果想每创建一个Pod时运行一个自定义的Initializers，可以这样做：

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
        - ""
        apiVersions:
        - v1
        resources:
        - pods
```

通过该配置创建资源对象InitializerConfiguration之后，就会在每个Pod的`metadata.initializers.pending`字段中添加`custom-pod-initializer`字段。该初始化控制器会定期扫描新的Pod，一旦在新的Pod的`pending`字段中检测到自己的名字，就会执行其逻辑，执行完逻辑后就会将`pending`字段下自己的名字删除。

只有在`pending`字段下的第一个Initializers可以对资源进行操作，当所有的Initializers执行完成，并且`pending`字段为空时，该对象就被认为初始化成功。

现在有一个问题：如果kube-apiserver不能显示这些资源，那么用户级的控制器是如何处理资源的呢？实际上，kube-apiserver暴露了一个`？includeUninitialized`查询参数，他会返回所有单位资源对象(包括未初始化的)。

## 控制循环

### 1. Deployments Controller

到这个阶段，Deployment已经记录在etcd中，并且所有初始化逻辑已经执行完成，接下来的阶段将会涉及到该资源所依赖的拓扑结构。在Kubernetes中，Deployment实际上是一系列`ReplicaSet`的集合，而ReplicaSet是一系列`Pod`的集合。Kubernetes依靠其内置控制器完成从HTTP请求按照层级结构依次创建资源的工作。

Kubernetes在整个系统中使用了大量的Controller，Controller是一个用于将系统状态从“当前状态”修正到“期望状态”的异步脚本。所有Controller都通过`kube-controller-manager`组件并行运行，每种Controller都负责一种具体的控制流程。首先介绍一下`Deployment Controller`：

将Deployment记录存储到etcd并初始化后，就可以通过kube-apiserver使其可见，然后`Deployment Controller`就会检测到它，控制器通过`Informer`注册一个创建事件的特定回调函数。

当Deployment第一次对外可见时，该Controller就会将该资源添加到内部工作队列，然后开始处理这个资源对象：

> 通过使用标签选择器查询kube-apiserver来检查该Deployment是否有与其关联的ReplicaSet或Pod对象。

在意识到没有与其关联的ReplicaSet或Pod记录后，Deployment Controller就会开始执行`弹性伸缩流程`:

> 创建ReplicaSet资源，为其分配一个标签选择器并将其版本号设置为1。

ReplicaSet的`PodSpec`字段从Deployment的manifest以及其他相关元数据中复制得来。有时Deployment记录在此之后也需要更新(例如设置了`process deadline`)。

当完成上述步骤后，该Deployment的`status`就会被更新，然后重新进入与之前相同的循环，等待Deployment与期望状态相匹配。由于Deployment Controller只关心ReplicaSet，因此需要通过`ReplicaSet Controller`来协调。

### 2. ReplicaSets Controller

前面的步骤中，Deployment Controller创建了一个ReplicaSet，但仍然没有Pod。这时就需要`ReplicaSet Controller`来继续工作。ReplicaSet Controller的工作是监视ReplicaSets及其相关资源（Pod）的生命周期。和其他大多数Controller一样，它通过触发某些事件的处理器来实现此目的。

当创建了ReplicaSet，ReplicaSet Controller检查新ReplicaSet的状态，并检查当前状态与期望状态的偏差，通过`调整Pod副本数`来达成期望状态。

Pod的创建也是批量进行的，从`SlowStartInitialBatchSize`开始，然后再每次成功的迭代中以一种`slow start`操作加倍。这样做的目的是在大量Pod启动失败时（例如，由于资源配额），可以减轻kube-apiserver被大量不必要的HTTP请求吞没的风险。如果创建失败，最好能够优雅地失败，这样对其他的系统组件影响最小。

Kubernetes通过`Owner References`（在子级资源的某个字段中引用其父级资源的ID）来构造严格的资源对象层级结构。这确保了一旦Controller管理的资源被删除（级联删除），子资源就会被垃圾收集器删除，同时还为父级资源提供了一种有效的方式来避免其竞争同一个子资源。

Owner References的另一个好处是：它是有状态的。如果任何Controller重启了，那么由于资源对象的拓扑状态与Controller无关，该操作不会影响到系统稳定运行。这种对资源隔离的重视也体现在Controller本身的设计中：Controller不能对自己没有明确拥有的资源进行操作，它们应该选择对资源的所有权，互不干涉，互不共享。

有时系统中也会出现孤儿（orphaned）资源，通常由以下两种途径产生：

- 父级资源被删除，但子级资源没有被删除
- 垃圾收集策略禁止收集子级资源

当发生这种情况时，Controller将会确保孤儿资源拥有新的`Owner`。多个父级资源可以相互竞争同一个孤儿资源，但只有一个会成功（其他父级资源会受到验证错误）。

### 3. Informers

某些Controller（例如RBAC授权器或Deployment Controller）需要先检索集群状态然后才能正常运行。拿RBAC授权器举例，当请求进入时，授权器会将用户的初始状态缓存下来，然后用它来检索与etcd中用户关联的所有角色（Role）和角色绑定（RoleBinding）。Controller是通过`Informer`机制来访问和修改这些资源对象的。

Informer是一种模式，它允许Controller查找缓存在本地内存中的数据（由Informer自己维护）并列出它们感兴趣的资源。

Informer在内部实现了大量对细节的处理逻辑（例如缓存），缓存很重要，因为它不仅可以减少对Kubernetes API的直接调用，同时也能减少Server和Controller的大量重复性工作。通过使用Informer，不同Controller之间以线程安全的方式进行交互，而不必担心多个线程访问相同资源时产生冲突。

### 4. Scheduler

当所有Controller正常运行后，etcd中就会保存一个Deployment、一个ReplicaSet和三个Pod资源记录，并且可以通过kube-apiserver查看。然而，这些Pod资源还处于`pending`状态，因为它们还没被调度到集群中合适的Node上运行，这个问题需要调度器来解决。

`Scheduler`作为一个独立的组件运行在集群的控制平面上，工作方式与其他Controller相同：监听实际并将系统状态调整到期望的状态。具体来说，Scheduler的作用是将待调度的Pod按照特定的算法和调度策略绑定到集群中某个合适的Node上，并将绑定信息写入etcd中（它会过滤其PodSpec中`NodeName`字段为空的Pod），默认调度算法的工作方式如下：

1. 当Scheduler启动时，会注册一个`默认的预选策略链`，这些预选策略会对备选节点进行评估，判断备选节点是否满足备选Pod的需求。例如，如果PodSpec字段限制了CPU和内存资源，那么当备选节点的资源容量不满足备选Pod的需求时，备选Pod就不会被调度到该节点上（资源容量=备选节点资源总量-节点中已存在Pod的所有容器的需求资源（CPU和内存）的总和）。
2. 一旦筛选出符合要求的候选节点，就会采用`优选策略`计算出每个候选节点的积分，然后对这些候选节点进行排序，积分最高者胜出。例如，为了在整个系统中分摊工作负载，这些优选策略会从候选节点中选出资源消耗最小的节点。每个节点通过优选策略都会计算出一个得分，计算各项得分，选出分值最大的节点作为优选的结果。

一旦找到了合适的节点，Scheduler就会创建一个`Binding`对象，该对象的`Name`和`UID`与Pod相匹配，并且其`ObjectReference`字段包含所选节点名称，然后通过`POST`请求发送给apiserver。

当kube-apiserver接收到该Binding对象时，注册表会将该对象反序列化并更新Pod资源中一下字段：

- 将`NodeName`值设置为ObjectReference中的NodeName
- 添加相关的注释
- 将`PodScheduled`的status设置为True。可以通过kubectl来查看
  
```
$ kubectl get <PODNAME> -o go-template='{{range .status.conditions}}{{if eq .type "PodScheduled"}}{{.status}}{{end}}{{end}}'
```

一旦Scheduler将Pod调度到某个节点上，该节点的`kubelet`就会接管该Pod并开始部署。

> 预选策略和优选策略都可以通过--policy-config-file参数来扩展，如果默认调度器不满足要求，还可以部署自定义的调度器。
> 
> 如果`podSpec.schedulerName`的值设置为其他的调度器，则Kubernetes会将该Pod的调度交给那个调度器。

## kubelet

### 1. Pod同步

现在所有的Controller都完成了工作，总结如下：

- HTTP请求通过了认证、授权和准入控制阶段
- 一个Deployment、一个ReplicaSet和三个Pod资源被持久化=存储到etcd中
- 然后运行了一系列Initializers
- 最后每个Pod都被调度到合适的节点

然而到目前为止，所有的状态变化仅仅只是针对保存在etcd中的资源记录，接下来的步骤涉及到运行在工作节点之间的Pod的分布状况，这是分布式系统的关键因素，这些任务都是由`kubelet`完成的。

在Kubernetes集群中，每个Node节点上都会启动一个kubelet服务进程，该进程用于处理Scheduler下发到本节点的任务，管理Pod的生命周期，包括挂载卷、容器日志记录、垃圾回收以及其他与Pod相关的事件。

如果换一种思维模式，可以把kubelet看成一种特殊的Controller，它每隔20s向kube-apiserver通过`NodeName`获取自身Node上要运行的Pod清单。一旦获取到这个清单，它就会通过与自己的内部缓存进行比较来检测新增加的Pod，如果有差异，就开始同步Pod列表。下面是同步过程：

1. 如果Pod正在创建，kubelet就会记录一些在Prometheus中用户追踪Pod启动延时的指标
2. 然后生成一个`PodStatus`对象，他表示Pod当前阶段的状态。Pod的状态（`Phase`）是Pod在其生命周期中最精简的概要，包括`Pending`、`Running`、`Succeed`、`Failed`和`Unknown`。状态的产生过程非常复杂，其背后的原理如下：
   - 首先执行一些列的同步处理器（`PodSyncHandlers`），每个处理器检查Pod是否应该运行在该节点上。当所有处理器都认为该Pod不应该运行在该节点上时，Pod的`Phase`值就会被设置为`Failed`，并且将该Pod从节点上驱逐出去。例如当你创建一个Job时，如果Pod的失败重试时间超过了`spec.activeDeadlineSeconds`设置的值，就会将Pod从该节点驱逐。
   - 接下来，Pod的`Phase`值由`init容器`和应用容器的状态来共同决定。因为目前容器还没有启动，容器被视为处于等待阶段，如果Pod中至少有一个容器处于等待阶段，则`Phase`值为`Pending`。
   - 最后，Pod的`Condition`字段由Pod内所有容器的状态决定，现在容器还没有被容器运行时创建，所以`PodReady的状态为False`。可以通过kubectl查看：
        ```
        $ kubectl get <PODNAME> -o go-template='{{range .status.conditions}}{{if eq .type "Ready"}}{{.status}}{{end}}{{end}}'
        ```

3. 生成PodStatus（Pod中的Status字段）后，kubelet就会将它发送到Pod的状态管理器，该管理器的任务是通过apiserver异步更新etcd中的记录。
4. 接下来运行一系列的准入控制器来确保该Pod是否具有相应的权限（包括强制执行AppArmor配置文件和NO_NEW_PRIVS），被准入控制器拒绝的Pod将一直保持`Pending`状态。
5. 如果kubelet启动时指定了`cgroups-per-qos`参数，kubelet就会为该Pod创建`cgroup`并进行相应的资源限制。这是为了更方便的对Pod进行服务质量（`Qos`）管理。
6. 然后为Pod创建相应的目录，包括Pod的目录（`/var/run/kubelet/pods/<podID>`），该Pod的卷目录（`<podDir>/volumes`）和该Pod的插件目录（`<podDir>/plugins`）。
7. 卷管理器会挂载`spec.Volumes`中定义的相关数据卷，然后等待是否挂载成功。根据挂载卷类型的不同，某些Pod可能需要等待更长的时间（例如NFS）。
8. 从apiserver中检索`spec.ImagePullSecrets`中定义的所有`Secret`，将其注入到容器中。
9. 最后通过容器运行时接口（`Container Runtime Interface（CRI）`）开始启动容器。

### 2. CRI与pause容器

容器是由容器运行时（例如`Docker`和`Rkt`）启动的。kubelet从1.5.0版本开始通过容器运行时接口与容器运行时交互。

第一次启动Pod时，kubelet会通过`Remote Procedure Command（RPC）`协议调用`RunPodSandbox`。`sandbox`用于描述一组容器，例如在Kubernetes中它表示的是Pod，而在`hypervisor`运行时中，sandbox指的就是虚拟机。

使用Docker容器运行时的时候，创建sandbox时首先创建的是`pause`容器。pause容器作为同一个Pod中所有其他容器的基础容器，它为Pod中每个业务容器提供了大量Pod级别资源，这些资源都是Linux命名空间（包括网络命名空间、IPC命名空间和PID命名空间）。

pause容器提供了一种方法来管理所有这些命名空间并允许业务容器共享它们，在同一个网络命名空间中的好处是：同一个Pod中的容器可以通过`localhost`进行通信。pause容器的第二个功能与PID命名空间的工作方式相关，在PID命名空间中，进程之间会形成一个树状结构，一旦某个子进程由于父进程的错误而变成了`孤儿进程`，其便会被`init`进程收养并最终回收资源。

### 3. CNI与Pod网络

现在Pod已经有了一个骨架：一个共享所有命名空间以允许业务容器在同一个Pod里进行通信的pause容器。但是网络还没有建立。

kubelet为Pod创建网络时，他会将创建网络的任务交给`CNI`插件。CNI表示容器网络接口（Container Network Interface），和容器运行时的运行方式类似，它也是一种抽象，允许不同的网络提供商为容器提供不同的网络实现。通过将json文件（默认在`/etc/cni/net.d`）中的数据传送到相关CNI二进制文件（默认在`/opt/cni/bin`）中，cni插件可以给pause容器配置网络，然后Pod中其他容器都可以使用pause容器的网络。下面是一个简单的示例配置文件：

```json
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

CNI插件还会通过`CNI_ARGS`环境变量为Pod指定其他的元数据，包括Pod名称和命名空间。

下面以`bridge`插件为例讲解以下步骤：
- 该插件首先会在根网络命名空间（也就是宿主机网络命名空间）中设置本地Linux网桥，以便为该主机上所有容器提供网络服务。
- 然后它会将一个网络接口（`Veth`设备的一端）插入到pause容器的网络命名空间中，并将另一端连接到网桥。
- 接下来json文件中指定的`IPAM`插件会为pause容器的网络接口分配一个IP并设置相应的路由，现在Pod就有了自己的IP。
- IPAM Plugin的工作方式和CNI Plugin类似：通过二进制文件调用并具有标准化接口，每一个IPAM Plugin必须要确定容器网络的IP、子网以及网关和路由，并将信息返回给CNI插件。最常见的IPAM Plugin是`host-local`，它从预定义的一组地址池中为容器分配IP地址。它将地址池信息以及分配信息保存在主机文件系统中，从未确保了同一台主机上每个容器的IP地址的唯一性。
- 最后kubelet会将集群内部`DNS`服务器的`ClusterIP`地址传给CNI插件，然后CNI插件将它们写到容器的`/etc/resolv.conf`文件中。

一旦完成上述步骤，CNI插件就会将操作的结果以json文件返回给kubelet。

### 4. 跨主机容器网络

通常情况下使用`overlay`网络来进行跨主机容器通信。最常用的网络插件是`Flannel`，包含`UDP`、`VXLAN`和`host-gw`三种网络实现方案。或者使用`Calico`网络插件。

### 5. 容器启动

所有网络配置完成之后，就可以启动容器了。

一旦sandbox完成初始化并处于`active`状态，kubelet就可以开始为其创建容器了，首先启动PodSpec中定一个init容器，然后再启动业务容器：

1. 首先拉取容器的镜像。如果是私有仓库的镜像，就会用PodSpec中指定的Secret来拉取该镜像。
2. 然后通过CRI接口创建容器。kubelet向PodSpec中填充了一个`ContainerConfig`数据结构（在其中定义了命令，镜像，标签，挂在卷，设备，环境变量等等），然后通过`protobufs`发送给CRI接口。对于Docker来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给`Dockerd`守护进程。在这个过程中，它会将一些元数据标签（例如容器类型、日志路径、sandbox ID等）添加到容器中。
3. 接下来使用CPU管理器来约束容器，使用`UpdateContainerResource`CRI方法将容器分配给本节点的CPU资源池。
4. 最后容器开始真正启动。
5. 如果Pod中定义了容器生命周期钩子（`Hook`），容器启动之后就会运行这些Hook。Hook的类型包括两种：`Exec`（执行一段命令）和`HTTP`（发送HTTP请求）。如果PostStart Hook启动时间过长、挂起或者失败，容器将永远不会变成`Running`状态。

## 总结

如果上面流程一切顺利，现在集群上会运行三个容器，所有的网络、数据卷和秘钥都通过CRI接口添加到容器中并配置成功。

上面整个创建Pod过程的流程图如下所示：

![整体流程](./images/kubelet-run.jpeg)