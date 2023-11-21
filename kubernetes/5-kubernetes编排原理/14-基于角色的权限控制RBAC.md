##**知识点**
- pass

##**Role和RoleBinding**
三个基本概念：
1. Role：角色。其实是一组规则，定义了一组对Kubernetes API对象的操作权限。
2. Subject：被作用者。用户或ServiceAccount。
3. RoleBinding：被作用者和角色之间的绑定关系。

Role是一个Kubernetes的API对象：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: example-role
  namespace: mynamespace
rules:
- apiGroup: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
这个Role对象指定了它能产生作用的Namespace是mynamespace，Kubernetes中没有指定Namespace时，使用的都是默认Namespace：default。

Role对象的rules字段就是定义的权限规则。这条规则的意思是，允许被作用者对mynamespace下的Pod对象进行GET、WATCH和LIST操作。

RoleBinding也是一个Kubernetes的API对象：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```
RoleBinding对象里定义了一个subjects字段，即被作用者。它的类型是User,名字时example-user。Role和RoleBinding对象都是Namespaced对象，它们对权限的限制规则仅在它们自己的Namespace里有效，roleRef也只能引用当前Namespace里的Role对象。

Role对象的rules字段可以进一步细化，可以只针对具体对象设置权限：
```yaml
rules:
- apiGroup: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```


##**ClusterRole和ClusterRoleBinding**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: example-clusterrole
rules:
- apiGroup: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```
ClusterRole和ClusterRoleBinding这对组合的用法与Role和RoleBinding用法完全一致，只是没有Namespace字段。

Kubernetes提供了4个预先定义好的ClusterRole供用户使用：
1. cluster-admin
2. admin
3. edit
4. view


##**ServiceAccount**
ServiceAccount是Kubernetes里的内置用户，它的API对象非常简单,一个简单的ServiceAccount对象只需要Name和Namespace了最基本的字段：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

通过RoleBinding来为ServiceAccount分配权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

在Pod的定义里声明ServiceAccountName：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
  namespace: mynamespace
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa
```
##**用户组Group**
一个ServiceAccount在Kubnernets里对应的用户名字是：
`system:serviceaccount:<ServiceAccount 名字>`
它对应的内置用户组的名字是：
`system:serviceaccounts:<Namespace 名字>`

在RoleBinding里使用用户组：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```
这就意味着Role的权限规则作用于mynamespace里所有的ServiceAccount。

下面这个例子意味着Role的权限规则作用域整个系统内所有的ServiceAccount：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

