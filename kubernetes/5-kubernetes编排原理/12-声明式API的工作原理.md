##**知识点**
- pass

##**声明式API的工作原理**
Kubernetes项目中，一个API对象在etcd里的完整路径是由`Group（API组）`、`Version（API版本）`和`Resource（API资源类型）`三个部分组成的。

Kubernetes核心API对象，比如Pod、Node等，不需要Group。对于这些API对象来说，Kubernetes直接在/api这个层级进行匹配。而对于CronJob这种非核心API对象来说，必须在/apis这个层级进行匹配，首先找到Group batch，然后匹配版本，最后匹配资源类型。

当发起创建CronJob的POST请求后，YAML文件的信息就提交给了API Server：
1. API Server首先过滤这个请求，完成前置性工作，比如授权、超时处理、审计等。
2. 请求进入`MUX`和`Routes`流程。MUX和Routes是API Server完成URL到Handler绑定的场所。API Server的Handler要做的就是按照匹配流程，找到CronJob类型定义。
3. API Server根据类型定义使用用户提交的YAML文件里的字段，创建一个CronJob对象。此过程中，API Server把用户提交的YAML文件转换成一个`Super Version`对象，它是该API资源类型所有版本的字段全集。
4. API Server先后进行Admission()和Validation()操作。上一节的Admission Controller和Initializer就属于Admission内容。Validation验证对象里各个字段是否合法。经过验证的API对象保存在API Server的`Registry`数据结构中。
5. API Server把经过验证的API对象转换成用户最初提交的版本，进行序列化操作，并调用etcd API进行保存。

##**添加Network资源类型**
现在要为Kubernetes添加一个名为Network的API资源类型，它的作用是，一旦用户创建了一个Network对象，那么Kubernetes使用这个Network对象定义的网络参数，调用真实网络插件，创建真正的网络。

这个Network对象的YAML文件example-network.yaml如下所示：
```yaml
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
```

上面这个YAML文件就是一个具体的自定义API资源的实例，也叫作CR（Custom Resource）。为了让Kubernetes能够认识这个CR，就需要让Kubernetes明白这个CR的宏观定义，CRD。
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```
在这个CRD中，指定了`group: samplecrd.k8s.io`,`version: v1`,耶指定了资源类型为Network，复数为networks。声明了scope是Namespaced，即Network是属于Namespace的对象。声明了宏观定义之后，还需要让Kubernetes认识这些字段的含义，在GOPATH下创建一个结构如下的项目：
```
$ tree $GOPATH/src/github.com/<your-name>/k8s-controller-custom-resource
.
├── controller.go
├── crd
|   └── network.yaml
├── example
|   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```
其中,`pkg/apis/samplecrd`为API组的名字，v1是版本，而v1下面的types.go定义了Network对象的完整描述。

`pkg/apis/samplecrd`下定义了`register.go`文件，用于放置后面要用到的全局变量：
```go
package samplecrd

const (
    GroupName = "samplecrd.k8s.io"
    Version   = "v1"
)
```
`pkg/apis/samplecrd/v1`下定义了`doc.go`文件,内容如下所示：
```go
// +k8s:deepcopy-gen=package
// +groupName=samplecrd.k8s.io

package v1
```
文件中包含了`+<tag_name>[=value]`格式的注释，这是Kubernetes进行代码生成要用到的Annotation风格的注释。

`+k8s:deepcopy-gen=package`意思是为整个v1包里所有类型定义自动生成Deepcopy方法，`+groupName=samplecrd.k8s.io`定义了这个包对应的API组的名字。

`types.go`文件作用是定义一个Network类型到底有哪些字段：
```go
package v1
...
// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interface=k8s.io/apimachinery/pkg/runtime.Object

type Network struct {
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec networkspec `json:"spec"`
}

type networkspec struct {
    Cidr    string `json:"cidr"`
    Gateway string `json:"gateway"`
}

// +k8s:deepcopy-gen:interface=k8s.io/apimachinery/pkg/runtime.Object

type NetworkList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`

    Items []Network `json:"items"`
}
```
可以看到Network类型定义方法和标准Kubernetes对象一样，都包含`TypeMeta(API元数据)`和`ObjectMeta(对象元数据)`字段。`Spec`字段就是需要自己定义的部分。

`NetworkList`类型，用来描述一组Network对象应该包含哪些字段。

`+genclient`意思是为下面的API资源类型生成对应的Client代码。`+genclient:noStatus`意思是这个API资源类型定义里没有Status字段。否则，生成的Client会自动带上UpdateStatus方法。

如果自定义类型包括Status字段，就不需要`+genclient:noStatus`：
```go
// +genclient

type Network struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec    NetworkSpec   `json:"spec"`
    Status  NetworkStatus `json:"status"`
}
```
`+genclient`只需要写在Network类型上，而不用写在NetworkList类型上。因为NetworkList类型是一个返回值类型，Network类型才是主类型。

`+k8s:deepcopy-gen:interface=k8s.io/apimachinery/pkg/runtime.Object`意思是生成Deepcopy时实现Kubernetes提供的`runtime.Object`接口。

“registry”的作用是注册一个类型给API Server。Network资源类型在服务器端的注册工作，API Server会自动完成。但是客户端也需要知道Network资源类型的定义，这就需要`pkg/apis/samplecrd/v1/registry.go`文件，其主要功能就是定义`addKnownTypes()`方法：
```go
package v1
...

func addKnowntypes(scheme *runtime.Scheme) error {
    scheme.AddKnownTypes(
        SchemeGroupVersion,
        &Network{},
        &NetworkList{},
    )

    metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
    return nil
}
```

接下来，使用Kubernetes的代码生成工具，为Network资源类型自动生成clientset、informer和lister。clientset就是操作Network资源类型的客户端,使用`k8s.io/code-generator`:
```
# 代码生成的工作目录，也就是项目路径
$ ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
# API Group
$ CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
$ CUSTOM_RESOURCE_VERSION="v1"

# 安装k8s.io/code-generator
$ go get -u k8s.io/code-generator/...
$ cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中pkg/client是生成目录，pkg/apis是类型定义目录
$ ./generate-group.sh all "ROOT_PACKAGE/pkg/client" "ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
```

生成后的项目目录结构如下：
```
├── controller.go
├── crd
|   └── network.yaml
├── example
|   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    |   └── samplecrd
    |       ├── register.go
    |       └── v1
    |           ├── doc.go
    |           ├── register.go
    |           └── types.go
    └── client
        ├── clientset
        ├── informers
        └── listers
```
此时可以使用`kubectl apply -f network.yaml`创建Network资源类型的CRD，然后使用`kubectl apply -f example-network.yaml`创建一个Network对象。



