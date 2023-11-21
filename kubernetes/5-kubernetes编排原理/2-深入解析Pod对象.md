##**知识点**
- Container是Pod属性里的一个普通字段。
- 凡是调度、网络、存储以及安全相关的属性，基本上是Pod级别的。
- 凡是跟容器的Linux Namespace相关的属性，一定是Pod级别的。

##**Pod中的重要字段**
###**NodeSelector**
NodeSelector是一个供用户将Pod与Node进行绑定的字段，用法如下：
```yaml
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    disktype: ssd
```
这样的配置意味着这个Pod只能在携带了disktype:ssd标签的节点上运行，否则将调度失败。

###**NodeName**
一旦这个字段被赋值，kubernetes将会认为这个Pod已调度，调度的结果就是赋值的节点名称。

###**HostAliases**
定义了Pod的hosts文件，用法如下：
```yaml
apiVersion: v1
kind: Pod
spec:
  hostAliases:
  - ip: "10.1.2.3"
  hostnames:
  - "foo.remote"
  - "bar.remote"
```
当这个Pod启动后，/etc/hosts文件的内容如下所示：
```
cat /etc/hosts
127.0.0.1   localhost
...
10.244.135.10   hostaliases-pod
10.1.2.3    foo.remote
10.1.2.3    bar.remote
```
在Kubernetes项目中，如果要设置hosts文件里的内容，一定要通过这种方式。如果直接修改了hosts文件，在Pod被删除重建后，kubelet会自动覆盖修改的内容。

###**shareProcessNamespace**
这个配置意味着这个Pod里的容器要共享PID Namespace。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```
在shell容器中执行ps命令，查看所有正在运行的进程：
```
kubectl attach -it nginx -c shell
/ # ps ax
PID     USER    TIMR    COMMAND
1       root    0:00    /pause
8       root    0:00    nginx: master process nginx -g daemon off;
14      101     0:00    nginx: worker process
15      root    0:00    sh
21      root    0:00    ps aux
```
在这个容器里，不仅可以看到它本身的ps ax指令，还可以看到nginx容器的进程，以及Infra容器的/pause进程。
###**hostNetwork、hostIPC、hostPID**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```
这个POD中，定义了hostNetwork、hostIPC、hostPID为true，即定义了共享宿主机的Network、IPC和PID Namespace。这就意味着，这个Pod里的所有容器都会直接使用宿主机的网络，直接与宿主机进行IPC通信，看到宿主机里正在运行的所有进程。
##**Pod中Containers字段中的属性**
###**ImagePullPolicy**
定义了镜像拉取策略。默认值为Always，即每次创建Pod都重新拉取一次镜像。另外，当镜像是类似于nginx或nginx:latest这样的名字时，ImagePullPolicy也会被认为是Always。
###**Lifecycle**
定义的是Container Lifecycle Hook，作用是在容器状态发生变化时触发一系列“钩子”。示例如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      prestop:
        exec:
          command: ["/usr/sbin/nginx", "-s", "quit"]
```
postStart指的是容器启动后立即执行一个指定操作。其定义的操作虽然是在Docker 容器ENTERYPOINT执行之后，但它并不严格保证顺序。也就是说，postStart启动时，ENTRYPOINT可能尚未结束。
preStop发生的时机是在容器被结束之前。preStop操作的执行是同步的，所以它会阻塞当前容器的结束流程，知道这个Hook定义的操作完成之后，才允许容器被结束。
##**Pod对象的生命周期**
Pod生命周期的变化主要体现在Pod API对象的Status部分。pod.status.phase就是Pod的当前状态，有以下几种可能：

1. Pending。这个状态意味着，Pod的YAML文件已经提交到了Kubernetes，API对象已经成功保存到了etcd中。但是这个Pod里有些容器因为某种原因不能顺利创建
2. Running。这个状态意味着，Pod已经调度成功，跟一个具体的节点绑定。它包含的容器已经全部创建成功，并且至少有一个正在运行。
3. Succeeded。这个状态意味着，Pod里的所有容器都正常运行完毕，并且已经退出了。这种情况在执行一次性任务时最常见。
4. Failed。这个状态意味着，Pod里至少有一个容器以不正常状态（非0的返回码）退出。
5. Unknown。这个状态意味着，Pod的状态不能持续地被kubelet汇报给kube-apiserver，可能是主从节点间的通信出了问题。
   
Pod的Status字段还可以细分出一组Conditions。细分状态主要包括：PodScheduled、Ready、Initialized以及Unschedulable。主要用来描述造成当前Status的原因。
Ready意味着Pod不仅已经正常启动（Running状态），而且可以对外提供服务了。