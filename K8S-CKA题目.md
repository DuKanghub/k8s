[toc]

**在考试时，只能用谷歌浏览器，并打开两个标签页，一个是考题的标签页，另一个是kubernetes官网标签页。**

## 以下 Daemonset yaml 中，哪些是正确的？（多选）

```
A. apiVersion: apps/v1 kind: DaemonSet metadata: name: fluentd-elasticsearch namespace: default labels: k8s-app: fluentd-logging spec: selector: matchLabels: name: fluentd-elasticsearch template: metadata: labels: name: fluentd-elasticsearch spec: containers: - name: fluentd-elasticsearch image: gcr.io/fluentd-elasticsearch/fluentd:v2.5.1 restartPolicy: Never
B. apiVersion: apps/v1 kind: DaemonSet metadata: name: fluentd-elasticsearch namespace: default labels: k8s-app: fluentd-logging spec: selector: matchLabels: name: fluentd-elasticsearch template: metadata: labels: name: fluentd-elasticsearch spec: containers: - name: fluentd-elasticsearch image: gcr.io/fluentd-elasticsearch/fluentd:v2.5.1 restartPolicy: Onfailure
C. apiVersion: apps/v1 kind: DaemonSet metadata: name: fluentd-elasticsearch namespace: default labels: k8s-app: fluentd-logging spec: selector: matchLabels: name: fluentd-elasticsearch template: metadata: labels: name: fluentd-elasticsearch spec: containers: - name: fluentd-elasticsearch image: gcr.io/fluentd-elasticsearch/fluentd:v2.5.1 restartPolicy: Always
D. apiVersion: apps/v1 kind: DaemonSet metadata: name: fluentd-elasticsearch namespace: default labels: k8s-app: fluentd-logging spec: selector: matchLabels: name: fluentd-elasticsearch template: metadata: labels: name: fluentd-elasticsearch spec: containers: - name: fluentd-elasticsearch image: gcr.io/fluentd-elasticsearch/fluentd:v2.5.1
```

### 答案：CD

### 答案解析：

https://kubernetes.io/查询daemonset的说明文档，

见以下链接：https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

> A Pod Template in a DaemonSet must have a RestartPolicy equal to Always, or be unspecified, which defaults to Always.
> Daemonset里的pod Template下必须有RestartPolicy，如果没指定，会默认为Always

- restartPolicy 字段，可选值为 Always、OnFailure 和 Never。默认为 Always。 一个Pod中可以有多个容器，restartPolicy适用于Pod 中的所有容器。restartPolicy作用是，让kubelet重启失败的容器。

- 另外Deployment、Statefulset的restartPolicy也必须为Always，保证pod异常退出，或者健康检查`livenessProbe`失败后由kubelet重启容器。https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/

- Job和CronJob是运行一次的pod，restartPolicy只能为OnFailure或Never，确保容器执行完成后不再重启。https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/

## 在Kubernetes PVC+PV体系下通过CSI实现的volume plugins动态创建pv到pv可被pod使用有哪些组件需要参与？

```
A. PersistentVolumeController + CSI-Provisoner + CSI controller plugin
B. AttachDetachController + CSI-Attacher + CSI controller plugin
C. Kubelet + CSI node plugin
```

### 答案：ABC

### 答案解析：

k8s中，利用PVC 描述Pod 所希望使用的持久化存储的大小，可读写权限等，一般由开发人员去创建；利用PV描述具体存储类型，存储地址，挂载目录等，一般由运维人员去提前创建。而不是直接在pod里写上volume的信息。一来可以使得开发运维职责分明，二来利用PVC、PV机制，可以很好扩展支持市面上不同的存储实现，如k8s v1.10版本对Local Persistent Volume的支持。

我们试着理一下Pod创建到volume可用的整体流程。

用户提交请求创建pod，PersistentVolumeController发现这个pod声明使用了PVC，那就会帮它找一个PV配对。

没有现成的PV，就去找对应的StorageClass，帮它新创建一个PV，然后和PVC完成绑定。

新创建的PV，还只是一个API 对象，需要经过“两阶段处理”，才能变成宿主机上的“持久化 Volume”真正被使用：

第一阶段由运行在master上的AttachDetachController负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘；

第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主控制循环。

完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。

k8s支持编写自己的存储插件FlexVolume 与 CSI。不管哪种方式，都需要经过“两阶段处理”，FlexVolume相比CSI局限性大，一般我们采用CSI方式对接存储。

CSI 插件体系的设计思想把这个Provision阶段（动态创建PV），以及 Kubernetes 里的一部分存储管理功能，从主干代码里剥离出来，做成了几个单独的组件。这些组件会通过 Watch API 监听 Kubernetes 里与存储相关的事件变化，比如 PVC 的创建，来执行具体的存储管理动作。![在这里插入图片描述](imgs/16e8e91b9e5e4ca7)上图中CSI这套存储插件体系中三个独立的外部组件（External Components），即：Driver Registrar、External Provisioner 和 External Attacher，对应的是从 Kubernetes 项目里面剥离出来的部分存储管理功能。

我们需要实现Custom Components这一个二进制，会以gRpc方式提供三个服务：CSI Identity、CSI Controller、CSI Node。

Driver Registrar 组件，负责将插件注册到 kubelet 里面；Driver Registrar调用CSI Identity 服务来获取插件信息；External Provisioner 组件监听APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，创建对应 PV；

External Attacher 组件负责Attach阶段。Mount阶段由kubelet里的VolumeManagerReconciler控制循环直接调用CSI Node服务完成。

两阶段完成后，kubelet将mount参数传递给docker，创建、启动容器。

整体流程如下图：![在这里插入图片描述](imgs/16e8e91bcfc97dd3)

## 通过单个命令创建一个deployment并暴露Service。deployment和Service名称为cka-1120，使用nginx镜像， deployment拥有2个pod

### 答案：

```bash
kubectl run cka-1120 --replicas=2 --expose=true --port=80 --image=nginx
```

### 答案解析：

官网中提供了详细的kubectl使用方法，位于REFERENCE-->>kubectl CLI-->>kubectl Commands标签下。即：https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#runkubectl run会创建deployment或者job来管理Pod，命令语法如下：

```bash
kubectl run NAME --image=image [--env="key=value"] [--port=port] [--replicas=replicas] [--dry-run=bool] [--overrides=inline-json] [--command] -- [COMMAND] [args...]
```

## 通过命令行，使用nginx镜像创建一个pod并手动调度到节点名为node1121节点上，Pod的名称为cka-1121

### 答案：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cka-1121
  labels:
    app: cka-1121
spec:
  containers:
  - name: cka-1121
    image: busybox
    command: ['sh', '-c', 'echo Hello CKA! && sleep 3600']
  nodeName: node1121
```

### 答案解析：

## 通过命令行，创建两个deployment，并配置pod在节点级别亲和与反亲和

```
通过命令行，创建两个deployment。
* 需要集群中有2个节点 ；
*  第1个deployment名称为cka-1122-01，使用nginx镜像，有2个pod，并配置该deployment自身的pod之间在节点级别反亲和；
* 第2个deployment名称为cka-1122-02，使用nginx镜像，有2个pod，并配置该deployment的pod与第1个deployment的pod在节点级别亲和；
最好提交最精简的deployment yaml，如果评论被限制，请提交反亲和性配置块yaml，也可多次评论提交
```

### 答案：

第一个deployment: cka-1122-01

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: cka-1122-01
  name: cka-1122-01
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cka-1122-01
  template:
    metadata:
      labels:
        app: cka-1122-01
    spec:
      containers:
      - image: nginx
        name: cka-1122-01  
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - cka-1122-01
              topologyKey: "kubernetes.io/hostname"
```

第二个deployment: cka-1122-02

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: cka-1122-02
  name: cka-1122-02
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cka-1122-02
  template:
    metadata:
      labels:
        app: cka-1122-02
    spec:
      containers:
      - image: nginx
        name: cka-1122-02
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cka-1122-01
            topologyKey: "kubernetes.io/hostname"
```

最终调度结果：

```bash
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
cka-1122-01-5df9bdf8c9-qwd2v    1/1         Running      0       8m     10.192.4.2     node-1
cka-1122-01-5df9bdf8c9-r4rhs    1/1      Running      0       8m     10.192.4.3     node-2  
cka-1122-02-749cd4b846-bjhzq     1/1      Running      0       10m    10.192.4.4    node-1
cka-1122-02-749cd4b846-rkgpo     1/1      Running      0       10m    10.192.4.5    node-2  
```

### 答案解析:

考点：k8s中的高级调度及用法。亲和性和反亲和性调度官方文档：https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

### 将 Pod 调度到特定的 Node 上：nodeSelector

nodeSelector是节点选择约束的最简单推荐形式。 nodeSelector是PodSpec下的一个字段。它指定键值对的映射。为了使Pod可以在节点上运行，该节点必须具有每个指定的键值对作为label。![在这里插入图片描述](imgs/16ead6497a732fbe)

> 语法格式：map[string]string  作用：
>
> – 匹配node.labels
>
> – 排除不包含nodeSelector中指定label的所有node
>
> – 匹配机制 —— 完全匹配

### nodeSelector 升级版：nodeAffinity

节点亲和性在概念上类似于nodeSelector，它可以根据节点上的标签来限制Pod可以被调度在哪些节点上。![在这里插入图片描述](imgs/16ead6499c8783a9)**红色框为硬性过滤：**排除不具备指定label的node；在预选阶段起作用；

**绿色框为软性评分：**不具备指定label的node打低分， 降低node被选中的几率；在优选阶段起作用；

#### 与nodeSelector关键差异

> – 引入运算符：In，NotIn （labelselector语法）
>
> – 支持枚举label可能的取值，如 zone in [az1, az2, az3...]
>
> – 支持硬性过滤和软性评分
>
> – 硬性过滤规则支持指定多条件之间的逻辑或运算
>
> – 软性评分规则支持 设置条件权重值

### 让某些 Pod 分布在同一组 Node 上：podAffinity

Pod亲和性和反亲和性可以基于已经在节点上运行的Pod上的标签而不是基于节点上的标签，来限制Pod调度的节点。

**规则的格式为：**

如果该X已经在运行一个或多个满足规则Y的Pod，则该Pod应该（或者在反亲和性的情况下不应该）在X中运行。

Y表示为LabelSelector。X是一个拓扑域，例如节点，机架，云提供者区域，云提供者区域等。![在这里插入图片描述](imgs/16ead64afb8b80fa)**红框硬性过滤：** 排除不具备指定pod的node组；在预选阶段起作用；

**绿框软性评分：** 不具备指定pod的node组打低分， 降低该组node被选中的几率；在优选阶段起作用；

#### 与nodeAffinity的关键差异

> – 定义在PodSpec中，亲和与反亲和规则具有对称性
>
> – labelSelector的匹配对象为Pod
>
> – 对node分组，依据label-key=topologyKey，每个labelvalue取值为一组
>
> – 硬性过滤规则，条件间只有逻辑与运算

### 避免某些 Pod 分布在同一组 Node 上：podAntiAffinity

![在这里插入图片描述](imgs/16ead64b215b0148)

#### 与podAffinity的差异



> – 匹配过程相同
>
> – 最终处理调度结果时取反
>
> 即
>
> – podAffinity中可调度节点，在podAntiAffinity中为不可调度
>
> – podAffinity中高分节点，在podAntiAffinity中为低分


## 通过命令行，创建1个deployment，副本数为3，镜像为nginx:latest。然后滚动升 级到nginx:1.9.1，再回滚到原来的版本

要求：Deployment的名称为cka-1125，贴出用到的相关命令。

最好附带创建的Deployment完整yaml，以及和升级回滚有关的命令。

### 答案：

先创建deployment，可以用命令创建：

```bash
kubectl run cka-1125  --image=nginx --replicas=3
```

也可以用以下yaml：cka-1125.yaml创建

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: cka-1125
  name: cka-1125
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cka-1125
  template:
    metadata:
      labels:
        app: cka-1125
    spec:
      containers:
      - image: nginx
        name: cka-1125
```

创建：

```bash
kubectl apply -f cka-1125.yaml
```

升级：

```bash
kubectl set image deploy/cka-1125 cka-1125=nginx:1.9.1 --record
deployment.extensions/cka-1125 image updated
```

回滚：

```bash
# 回滚到上一个版本
kubectl rollout undo deploy/cka-1125
# 回滚到指定版本
kubectl rollout undo deploy/cka-1125 --to-revision=2
```

### 答案解析

**set image**命令格式如下：

```bash
kubectl set image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N [--record]
```

--record指定，在annotation中记录当前的kubectl命令。 如果设置为false，则不记录命令。 如果设置为true，则记录命令。 默认为false。

```bash
[root@liabio test]# kubectl set image deploy/cka-1125 cka-1125=nginx:1.9.1 --record
deployment.extensions/cka-1125 image updated
[root@liabio test]# 
[root@liabio test]# kubectl rollout history deploy/cka-1125 
deployment.extensions/cka-1125 
REVISION  CHANGE-CAUSE
3         <none>
4         kubectl set image deploy/cka-1125 cka-1125=nginx:1.9.1 --record=true
```

像上面这样，CHANGE-CAUSE中会有升级命令。

`set image`命令可以对：`pod (po), replicationcontroller (rc), deployment (deploy), daemonset (ds), replicaset (rs)，statefulset(sts)`进行操作。

**rollout**命令

**官方文档：**https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout

可以管理`deployments、daemonsets、statefulsets`资源的回滚

查询升级历史

```bash
[root@liabio test]# kubectl rollout history deploy/cka-1125 
deployment.extensions/cka-1125 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

查看指定版本的详细信息

```bash
kubectl rollout history deploy/cka-1125 --revision=3 -o=yaml
```

回滚到上一个版本：

```bash
[root@liabio test]# kubectl rollout undo deploy/cka-1125 
deployment.extensions/cka-1125 rolled back
```

回滚到指定版本：

```bash
[root@liabio test]# kubectl rollout undo deploy/cka-1125 --to-revision=3
deployment.extensions/cka-1125 rolled back
```

**其他roll子命令**

restart：资源将重新启动；status：展示回滚状态；resume：恢复被暂停的资源。控制器不会控制被暂停的资源。通过恢复资源，可以让控制器再次控制。 resume仅对deployment支持。pause：控制器不会控制被暂停的资源。 使用`kubectl rollout resume`来恢复暂停的资源。 当前，只有deployment支持被暂停。

**滚动更新策略**

**官网文档：**https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

```yaml
minReadySeconds: 5
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

**minReadySeconds**Kubernetes在等待设置的时间后才进行升级如果没有设置该值，Kubernetes会假设该容器启动起来后就提供服务了如果没有设置该值，在某些极端情况下可能会造成服务不能正常运行

**maxSurge**

控制滚动更新过程中副本总数超过DESIRED的上限。maxSurge可以是具体的整数，也可以是百分比，向上取整。maxSurge默认值为25%。

例如DESIRED为10，那么副本总数的最大值为roundUp(10 + 10*25%)=13,所以CURRENT为13。

**maxUnavaible**控制滚动更新过程中，不可用副本占DESIRED的最大比例。maxUnavailable可以是具体的整数，也可以是百分之百，向下取整。默认值为25%。

例如DESIRED为10，那么可用的副本数至少要为 `10-roundDown(10*25%)=8`所以AVAILABLE为8。

**maxSurge越大，初始创建的新副本数量就越多；maxUnavailable越大，初始销毁的旧副本数目就越多。**

## 提供一个pod的yaml，要求添加Init Container，Init Container的作用是创建一个空文件，pod的Containers判断文件是否存在，不存在则退出

### 答案：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: cka-1126
  name: cka-1126
spec:
  initContainers:
  - image: busybox
    name: init-c
    command: ['sh', '-c', 'touch /tmp/cka-1126']
    volumeMounts:
    - name: workdir
      mountPath: "/tmp"
  containers:
  - image: busybox
    name: cka-1126
    command: ['sh', '-c', 'ls /tmp/cka-1126 && sleep 3600 || exit 1']
    volumeMounts:
    - name: workdir
      mountPath: "/tmp"
  volumes:
  - name: workdir
    emptyDir: {}
```

主Container的command就是判断文件是否存在，存在则不退出，不存在则退出；也可以用以下if判断：

```bash
command: ['sh', '-c', 'if [ -e /tmp/cka-1126 ];then echo "file exits";else echo "file not exits" && exit 1;fi']
```

## deployment升级与回滚

暂停deployment

```bash
kubectl rollout pause deploy nginx
```

恢复deployment

```bash
kubectl rollout resume deploy nginx
```

查询升级状态

```bash
kubectl rollout status deploy nginx
```

查询升级历史

```bash
kubectl rollout history deploy nginx
kubectl rollout history deploy nginx --revision=2
```

回滚

```bash
kubectl rollout undo deploy nginx --to-revivsion=2
# 如果不指定版本，默认回滚到上一个版本
```

## 应用弹性伸缩

```bash
kubectl scale deploy nginx --replicas=10
```

对接了heapster，和HPA联动后

```bash
kubectl autoscale deploy nginx --min=10 --max=15 --cpu-percent=80
```

对接`custom.metrics.k8s.io`后，可根据自定义指标伸缩

