## **知识点**
- 在Deployment中，restartPolicy只允许设置为Always；在Job中，restartPolicy只允许设置为Never或者OnFailure。
- Job Controller控制的对象是Pod。
- CronJob是一个Job对象的控制器。

## **Job**
离线业务，也称Batch Job，这种业务在计算完成后直接退出，不能使用Deployment来管理这种业务，而应该使用Job对象：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      contianers:
      - name: pi
        image: resourcer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l"]
      restartPolicy: Never
      activeDeadlineSeconds: 100
  backoffLimit: 4
```
这个Job定义了一个计算π值的容器，跟其他控制器不同，Job对象不要求定义一个`sepc.selector`来描述要控制哪些Pod。

如果离线作业失败，会不断有新的Pod被创建出来，`backoffLimit`字段定义了重试次数为4，默认为6。Job Controller重新创建Pod的间隔是呈指数级增长的。如果定义了`restartPolicy=OnFailure`，则不会创建新Pod，而是会不断重启Pod。

`activeDeadlineSeconds`字段意味着一旦运行超过Deadline时间，也就是100s，这个Job的所有Pod都会终止。

`spec.parallelism`定义的是一个Job在任意时间最多可以启动多少个Pod同时运行。

`sepc.completions`定义的是Job至少要完成的Pod数目，即Job的最小完成数。

## **Job对象使用方法**
### **第一种：外部管理器+Job模板**
特定用法为：把Job的YAML文件定义为一个模板，然后用一个外部工具控制这些模板来生成Pod。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    medadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```
控制以上Job时，需要注意以下两点：
1. 创建Job时，替换$ITEM这样的变量。
2. 所有来自同一个模板的Job，都有一个`jobgroup: jobexample`标签。也就是说，这一组Job使用这样一个相同的标识。

可以使用这样一句SHELL替换$ITEM：
```
$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
$ kubectl create -f ./jobs
```
### **第二种：拥有固定任务数的并行Job**
这种模式下，只关心最后是否有指定数目的任务成功退出，而不关心执行时的并行度：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  templete:
    medatada:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: myrepo/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: Never
```
在这个Job中，任务总数为8，意味着总共有8个任务会被逐一房屋工作队列中。在这个实例中，使用Kubernetes集群中的RabbitMQ作为工作队列，所以需要定义BROKER_URL作为消费者。

一旦创建了这个Job，它会以并发度为2的方式，每两个Pod一组创建出8个Pod。每个Pod都会连接BROKER_URL,从RabbitMQ中取任务。这个Pod里的执行逻辑可以使用以下伪代码来表示：
```go
queue := newQueue($BROKER_URL, $QUEUE)
task := queue.pop()
process(task)
exit
```
### **第三种：拥有固定并行度的并行Job**
这种模式下，指定并行度，但不设定固定的completions值。此时，必须自己决定何时启动新Pod，何时Job才算执行完成。此时不仅需要一个工作队列来负责任务的分发，还需要能够判断工作队列为空：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  templete:
    medatada:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job2
      restartPolicy: OnFailure
```

此时不需要执行`completions`值，Pod的逻辑如下：
```go
for !queue.IsEmpty($BROKER_URL, $QUEUE){
    task := queue.pop()
    process(task)
}
print("Queue empty, exiting")
exit
```
## **CronJob**
CronJob描述的是定时任务，它是一个Job对象的控制器：
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - data; echo Hello form the Kubernetes cluster
          restartPolicy: OnFailure
```
CronJob创建和删除Job的依据是`schedule`字段，它是一个标准的UNIX CRON格式的表达式。

由于定时任务有特殊性，可能上一个Job还没执行完，另一个新的Job就产生了。可以通过`spec.concurrencyPolicy`字段来定义具体的处理策略：
1. concurrencyPolicy=Allow， 默认情况，意味着允许这些Pod同时存在。
2. concurrencyPolicy=Forbid， 意味着不会创建新Pod，该创建周期被跳过。
3. concurrencyPolicy=Replace，意味着新的Job会替换旧的、未完成的Job。

如果一个Job创建失败，这次创建就会被标记为miss。在指定时间窗口内miss数目达到100时，CronJob会停止创建这个Job。时间窗口可以通过`spec.startingDeadlineSeconds`字段指定。