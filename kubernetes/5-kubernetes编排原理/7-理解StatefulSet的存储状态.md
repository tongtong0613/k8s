## **知识点**
- PVC是一种特殊的Volume，只不过PVC是什么类型的Volume，要和某个PV绑定之后才能知道。
- StatefulSet控制器直接管理Pod，Deployment控制器直接管理ReplicaSet。

## **PVC和PV**
Kubernetes引入一组叫做PVC和PV的API对象，降低用户声明和使用PV的门槛。有了PVC之后，想要使用一个Volume，只需以下两步：
**第一步：定义一个PVC，声明想要的Volume属性**
```yaml
apiVersion: v1
kind: PresistentVolumeClaim
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
这个PVC对象里，不需任何Volume细节的字段，只有描述性的属性和定义。
**第二步：定义一个Pod，在Pod中声明使用这个PVC**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```
上面这个Pod的Volumes定义中，只需要声明它的类型是persistentVolumeClaim，然后指定PVC名字，完全无需关心Volume本身的定义。

只要创建这个PVC对象，Kubernetes会自动为其绑定一个符合条件的Volume。而这些Volume来自于运维人员维护的PV对象：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-volume
  labels:
   type: local
spec:
  capacity:
    storage: 10Gi
  rbd:
    monitors:
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
    imageformat: "2"
    imagefeatures: "layering"
```
PVC和PV的设计，类似于接口和实现的思想。开发者只要知道并会使用接口（PVC），运维人员负责给接口绑定具体的实现（PV）。

## **StatefulSet使用PVC**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchlabels:
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
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplate:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          starage: 1Gi
```
这个StatefulSet额外添加了一个volumeClaimTemplate字段，它和Deployment里Pod模板的作用类似。也就是说，凡是被这个StatefulSet管理的Pod，都会声明这个PVC。最重要的是，**这个PVC的名字会被分配一个与这个Pod完全一致的编号**。

创建这个StatefulSet后，集群里会出现两个PVC：www-web-0和www-web-1。这些PVC命名遵循以下规则：
```
<PVC名字>-<StatefulSet名字>-<编号>
```
这个StatefulSet创建出来的所有Pod都会声明使用编号的PVC。比如，在名为web-0的Pod的Volumes字段，他会声明使用名为www-web-0的PVC，从而挂载这个PVC所绑定的PV。

如果删除这两个Pod，这两个Pod会按照编号的顺序重新创建。原先与名为web-0的Pod绑定的PV，在这个Pod被重新创建出来之后，依旧与其绑定在一起。

## **StatefulSet工作原理**
**首先，StatefulSet的控制器直接管理的是Pod**。因为StatefulSet里不同Pod实例不再像ReplicaSet中那样是完全一样的。比如，每个Pod的hostname、名字等都不同，都携带了编号。而StatefulSet通过在Pod的名字里加上编号来区分这些实例。

**其次，Kubernetes通过Headless Service为这些有编号的Pod，在DNS服务器中生成带有相同编号的DNS记录**。只要StatefulSet能够保证这些Pod名字里的编号不变，那么Service里类似于web-0.nginx.default.svc.cluster.local这样的DNS记录就不会变，而这条记录解析出来的IP地址，会随着Pod的删除核重建而自动更新。

**最后，StatefulSet还未没一个Pod分配并创建一个相同编号的PVC**。这样，Kubernetes就可以通过Persistent Volume机制为这个PVC绑定对应的PV，从而保证了每一个Pod都拥有一个独立的Volume。