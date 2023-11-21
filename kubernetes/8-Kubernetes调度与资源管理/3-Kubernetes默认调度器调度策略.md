## **知识点**
- pass

## **Predicate**

开始调度一个Pod时，Kubernetes调度器会启动16个Goroutine来并发执行预选策略。

1. GeneralPredicate：最基础的调度策略
   - PodFitsResources：计算宿主机的CPU和内存资源是否够用
   - PodFitsHost：检查宿主机名字是否和Pod的spec.nodeName一致
   - PodFitsHostPorts：检查Pod申请的宿主机端口是否和已使用端口冲突
   - PodMatchNodeSelector：检查nodeSelector或nodeAffinity指定的节点是否与待考察节点匹配

2. 与Volume相关的过滤规则
    - NoDiskConflict：检查多个Pod声明挂载的PV是否冲突
    - MaxPDVolumeCountPredicate：检查一个节点上某个类型PV是否超过一定数目
    - VolumeZonePredicate：检查PV的Zone标签是否与待考察节点zone标签匹配
    - VolumeBindingPredicate：检查Pod对应的PV的nodeAffinity字段是否和某个节点的标签匹配
  
3. 宿主机相关的过滤规则
   - PodToleratesNodeTaint：检查节点的污点机制，只有当Pod的Toleration字段与Node的Taint字段匹配时，Pod才能调度到该节点
   - NodeMemoryPressurePredicate：检查当前节点内存是否足够

4. Pod相关的过滤规则
    - PodAffinityPredicate：检查待调度Pod与节点上已有Pod的亲密与反亲密关系

## **Priority**

- LeastRequestedPriority：选择空闲资源最多的宿主机
- BalancedResourceAllocation：选择分配后使节点分配最均衡的节点
- ImageLocalityPriority：待调度Pod需要很大镜像并且镜像已经存在于某些节点，那些节点得分会高