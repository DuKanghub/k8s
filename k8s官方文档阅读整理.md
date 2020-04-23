<font size=4>

# k8s官方文档阅读整理
## 2019-07-08
1. 在集群外的电脑上使用`kubectl`远程控制k8s集群
```bash
# 将管理员kubeconfig文件从master节点复制到您的工作站
scp root@<master ip>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```
2. 将`kube-apiserver`代理到`localhost`
```bash
scp root@<master ip>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf proxy
# 您现在可以在本地访问API服务器 
curl http://localhost:8001/api/v1
```
3. 节点状态查看
```bash
kubectl describe node <insert-node-name-here>
```
地址信息(address)：
- `HostName`: 节点主机名，可通过，`kubelet` `--hostname-override`参数覆盖
- `ExternalIP`: 外部节点IP
- `InternalIP`: 通常仅在群集内可路由的节点的IP地址

状态信息(conditions)：
- `OutOfDisk`: 如果节点上的可用空间不足以添加新的pod，则为true，否则为false
- `Ready`: 如果节点是健康的并准备好接受pod为true,如果节点不健康且不接受pod，并且Unknown(节点控制器在最后一次没有从节点听到node-monitor-grace-period（默认为40秒）),则为false
- `MemoryPressure`: 如果节点存储器上存在压力,则为true
- `PIDPressure`: 如果节点进程数太多，则为true
- `DiskPressure`: 如果磁盘容量低，则为true
- `NetworkUnavailable`: 如果节点没有正确配置网络，则为true
如果`Ready`状态长时间处于`Unknown`或`False`，超过5分钟，`kube-controller-manager`会删除此节点上的所有pod。但是如果apiserver无法与节点的kubelet通信，将不能删除pod。这个操作在1.5前是强制的，1.5后(含1.5)不是强制的。

## 2020-04-15

### 1. 集群的当前状态是否符合其预期状态由谁保障？

答：Pod 生命周期事件生成器([PLEG](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md))，这其实是执行者，决定者是kube-controller-manager

问：Pod 生命周期事件生成器([PLEG](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/pod-lifecycle-event-generator.md))又是属于哪个组件？

答：kubelet

### 2. k8s基本对象有哪些？

答：Pod，Service,volume,Namespace

### 3. 被称为Controller的对象的高级抽象

答：Deployment,DaemonSet,StatefulSet,ReplicaSet,Job

### 4. Kube-controller-manager有哪些组件？

答：从字面意思是controller管理者，那就意味着这个二进制文件里包含多个controller。

这些控制器包括:

- 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应。
- 副本控制器（Replication Controller）: 负责为系统中的每个副本控制器对象维护正确数量的 Pod。
- 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)。
- 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

### 5. 筛选pod的几种方式

使用label

```bash
kubectl get pods -l environment=product
```

使用field-selector

```bash
kubectl get pods --field-selector status.parse=Running
```

### 6. 驱逐pod的过程是怎样的？

当人为删除node或驱逐pod，或节点状态(Ready)处于`Unknown`或`False`的时间超过controller-manager配置的`pod-eviction-timeout`(默认是5分钟)，节点的所有pod都会被controller-manager中的node-controller标记为Terminating，这个状态变更会提交到kube-apiserver，而kubelet接受到这个信息后，会执行这个Terminate的工作。所以，如果此时kube-apiserver无法与kubelet通信，此node上的pod并不会被驱逐。

**注意：**这个驱逐在1.5前的版本是强制的，但1.5后的版本不会强制，需要自己手动执行

### 7. 如何标记一个node为不可调度

标记一个节点为不可调度的将防止新建 pods 调度到那个节点之上，但不会影响任何已经在它之上的 pods。这是重启节点等操作之前的一个有用的准备步骤

```bash
kubectl cordon $NODENAME
```

## 2020-04-16

### 1. 如何查看一个已经崩溃的容器日志

```bash
kubectl logs ${POD_NAME} --previous
# 思考：k8s中pod崩溃会再起一个pod，而新pod的名字和旧pod的名字是不一样的，这条是否适用呢？如果适用，POD_NAME是指现在运行的POD_NAME还是之前的。
```

如果pod中有多个容器，应加`-c ${CONTAINER_NAME}`查看指定容器日志，否则，将查看默认的容器日志，即pod第一个创建的容器。

### 2. kubelet的垃圾回收策略

节点运行久了后，可能会产生大量的镜像残留和容器残留，这点kubelet是怎样做的呢？

Kubelet 将每分钟对容器执行一次垃圾回收，每五分钟对镜像执行一次垃圾回收。

不建议使用外部垃圾收集工具，因为这些工具可能会删除原本期望存在的容器进而破坏 kubelet 的行为。

#### 镜像回收

Kubernetes 借助于 cadvisor 通过 imageManager 来管理所有镜像的生命周期。

镜像垃圾回收策略只考虑两个因素：`HighThresholdPercent` 和 `LowThresholdPercent`。

磁盘使用率超过上限阈值（HighThresholdPercent）将触发垃圾回收。

垃圾回收将删除最近最少使用的镜像，直到磁盘使用率满足下限阈值（LowThresholdPercent）。

**也就是说，上面说的每5分钟对镜像执行一次垃圾回收，只是执行一次检查，磁盘使用超过HighThresholdPercent或LowThresholdPercent才会触发真正的回收。**

具体回收策略查看：https://kubernetes.io/zh/docs/concepts/cluster-administration/kubelet-garbage-collection/

### 3. 访问apiserver的三步

- 身份认证

  - 身份验证步骤的输入是整个 HTTP 请求，但是，它通常只检查标头和 / 或客户端证书
  - 身份验证模块包括客户端证书、密码and Plain Tokens, Bootstrap Tokens, and JWT Tokens (used for service accounts)
  - 可以指定多个身份验证模块，在这种情况下，按顺序尝试每个模块，直到其中一个成功
  - 如果请求不能通过身份验证，它将被 HTTP状态码401拒绝

- 授权

  - 请求必须包括请求者的用户名、请求的操作和受操作影响的对象
  - 支持多个授权模块，如 ABAC 模式、 RBAC 模式和 Webhook 模式
  - 如果所有模块都拒绝请求，那么请求将被拒绝(HTTP状态码403)

- 准入控制

  - 在请求被授权之后，如果是写入操作，它还将进入[准入控制](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)步骤

  - 与身份验证和授权模块不同，如果任何接纳控制器模块拒绝，那么请求将立即被拒绝

## 2020-04-17

### 1. cordon/drain/delete node的区别

#### cordon

> cordon是将节点设为不可调度，也就是说新起的pod不会调度到此节点运行，此节点上的原pod依旧可以提供服务。

思考：那此节点的原pod挂掉后，重新起的pod还会调到此节点吗？应该是不会的。

命令用法

```bash
# 设为不可调度
kubectl cordon node $NODE_NAME
# 恢复调度
kubectl uncordon node $NODE_NAME
```

#### drain

> drain是驱逐节点上的pod，请注意daemonset或statefulset可能无法驱逐，需要忽悠错误强制驱逐。

命令用法

```bash
kubectl drain node $NODE_NAME
# 如果有daemonset，需忽略错误
kubectl drain node $NODE_NAME --force --ignore-daemonsets
```

#### delete

> delete就是简单粗暴的删除节点

### 2. 节点安全维护

先将节点设置为不可调度

```bash
kubectl cordon node $NODE_NAME
```

再驱逐此节点上的pod

```bash
kubectl drain node $NODE_NAME --force --ignore-daemonsets
```

对节点进行维护，比如升级内核或docker等操作

维护完后，将节点重新恢复为可调度

```bash
kubectl uncordon node $NODE_NAME
```

**特别说明：**

对于statefulset创建的Pod，kubectl drain的说明如下：

kubectl drain操作会将相应节点上的旧Pod删除，并在可调度节点上面起一个对应的Pod。当旧Pod没有被正常删除的情况下，新Pod不会起来。例如：旧Pod一直处于`Terminating`状态。

对应的解决方式是通过重启相应节点的kubelet，或者强制删除该Pod。

示例：

```bash
# 重启发生`Terminating`节点的kubelet
systemctl restart kubelet

# 强制删除`Terminating`状态的Pod
kubectl delete pod <PodName> --namespace=<Namespace> --force --grace-period=0
```

### 3. ReplicaSet和ReplicationController的区别

ReplicaSet是新一代的RelicationController，*ReplicaSet* 和 [*Replication Controller*](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 的唯一区别是选择器的支持。ReplicaSet 支持新的基于集合的选择器需求，这在[标签用户指南](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)中有描述。而 Replication Controller 仅支持基于相等选择器的需求。

不过我们一般不直接使用ReplicaSet，而是直接使用Deployment，他会管理ReplicaSet

