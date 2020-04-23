[toc]

## 22. 集群使用-taints与tolerations

### 22.1 概念说明

Taint（污点）和 Toleration（容忍）可以作用于 node 和 pod 上，其目的是优化 pod 在集群间的调度，这跟节点亲和性类似，只不过它们作用的方式相反，具有 taint 的 node 和 pod 是互斥关系，而具有节点亲和性关系的 node 和 pod 是相吸的。另外还有可以给 node 节点设置 label，通过给 pod 设置 nodeSelector 将 pod 调度到具有匹配标签的节点上。

Taint 和 toleration 相互配合，可以用来避免 pod 被分配到不合适的节点上。每个节点上都可以应用一个或多个 taint ，这表示对于那些不能容忍这些 taint 的 pod，是不会被该节点接受的。如果将 toleration 应用于 pod 上，则表示这些 pod 可以（但不要求）被调度到具有相应 taint 的节点上。

### 22.2 node节点设置污点

设置污点

```bash
# 样式
kubectl taint nodes node1 key1=value1:NoSchedule
# 实例，拿前面给lb打的标签
kubectl taint nodes lb01 node-role.kubernetes.io/ingress:NoSchedule
# node-role.kubernetes.io/ingress是key，这里没有value，NoSchedule是effort
```

| Effort           | 说明                                  |
| ---------------- | ------------------------------------- |
| NoSchedule       | 一定不能被调度                        |
| PreferNoSchedule | 尽量不要调度                          |
| NoExecute        | 不仅不会调度, 还会驱逐Node上已有的Pod |

注意：节点设置污点后，除非有匹配的容忍，否则新pod不会往此节点调度。对于在设置taints前已经存在的pod,如果effort为NoExecute，则Pod会被驱除，反之，Pod仍可停留在此node上

查看节点taints

```bash
# kubectl describe node lb01 |grep Taints
Taints:             node-role.kubernetes.io/ingress:NoSchedule 
```

删除taint

```bash
# 样式
kubectl taint node node1 key1:NoSchedule-  # 这里的key可以不用指定value
kubectl taint node node1 key1:NoExecute-
kubectl taint node node1 key1-             # 删除指定key所有的effect
kubectl taint node node1 key2:NoSchedule-
```

### 22.3 Pod设置容忍

拿上面打了污点的lb01节点，如下设置容忍的Pod将可以调度到此节点(不是一定会调到此节点)，注意operator是Equal(后面再详说)

```yaml
tolerations:
- key: "node-role.kubernetes.io/ingress"
  operator: Equal
  effect: NoSchedule
```

再看一个容忍例子

```yaml
tolerations:
- operator: Exists
  effect: NoSchedule
```

`taint` 和 `toleration` 要匹配上，需要满足两者的 `keys` 和 `effects` 是一致的，且：

- 当 `operator` 是 `Exists` （意味着不用指定 `value` 的内容）时，或者
- 当 `operator` 是 `Equal` 时 `values` 也相同

注1： `operator` 默认值是 `Equal` 如果不指定的话

注2: 留意下面 2 个使用 `Exists` 的特例

- key 为空且 `operator` 是 `Exists` 时，将匹配所有的 `keys`, `values` 和 `effects` ，这表明可以 `tolerate`所有的 `taint` 

  ```yaml
  tolerations:
  - operator: Exists
  ```

  这个常用于部署daemonset到所有节点上，比如用pod部署的flannel

-  `effect` 为空将匹配这个 `key` 对应的所有的 `effects` 

  ```yaml
  tolerations:
  - operator: Exists
    key: "node-role.kubernetes.io/ingress"
  ```

  

### 22.4 注意事项

可以在同一个 node 上使用多个 `taints` ，也可以在同一个 pod 上使用多个 `tolerations` ，而 k8s 在处理 `taints and tolerations` 时类似一个过滤器： 

- 对比一个 node 上所有的 `taints`
- 忽略掉和 pod 中 `toleration` 匹配的 `taints`
- 遗留下来未被忽略掉的所有 `taints` 将对 pod 产生 `effect`

尤其是：

- 至少有 1 个未被忽略的 `taint` 且 `effect` 是 `NoSchedule` 时，则 k8s 不会将该 pod 调度到这个 node 上
- 不满足上述场景，但至少有 1 个未被忽略的 `taint` 且 `effect` 是 `PreferNoSchedule` 时，则 k8s 将尝试不把该 pod 调度到这个 node 上
- 至少有 1 个未被忽略的 `taint` 且 `effect` 是 `NoExecute` 时，则 k8s 会立即将该 pod 从该 node 上驱逐（如果已经在该 node 上运行），或着不会将该 pod 调度到这个 node 上（如果还没在这个 node 上运行）

**实例**，有下述 node 和 pod 的定义：

```bash
kubectl taint nodes tvm-04 key1=value1:NoSchedule
kubectl taint nodes tvm-04 key1=value1:NoExecute
kubectl taint nodes tvm-04 key2=value2:NoSchedule
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

上述场景中，

- 该 pod 不会调度到 node tvm-04上，因为第 3 个 taint 不满足
- 如果该 pod 已经在该 node tvm-04上运行，则不会被驱逐

通常而言，不能 `tolerate` 一个 `effect` 是 `NoExecute` 的 pod 将被立即驱逐，但是，通过指定可选的字段 `tolerationSeconds` 则可以规定该 pod 延迟到一个时间段后再被驱逐，例如：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

也就是说，在 3600 秒后将被驱逐。但是，如果在这个时间点前移除了相关的 `taint` 则也不会被驱逐
注3：关于被驱逐，如果该 pod 没有其他地方可以被调度，也不会被驱逐出去（个人实验结果，请自行验证）

