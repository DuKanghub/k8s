[toc]

------

**参考地址：**

- 部署文件 Github 地址：https://github.com/my-dlq/blog-example/tree/master/kubernetes/kubernetes-dashboard2.0.0-beta4-deploy

**系统环境：**

- Kubernetes 版本：1.15.0

- kubernetes-dashboard 版本：v2.0.0-beta4

  当前最新的版本是beta6，完全支持1.16，不一定完全支持1.15，所以这里选择beta4

## 一、简介

​    Kubernetes Dashboard 是 Kubernetes 集群的基于 Web 的通用 UI。它允许用户管理在群集中运行的应用程序并对其进行故障排除，以及管理群集本身。这个项目在 Github 已经有半年多不更新了，最近推出了 v2.0.0-beta4 版本，这里在 Kubernetes 中部署一下，尝试看看新版本咋样。

## 二、兼容性

| Kubernetes版本 | 1.9  | 1.10 | 1.11 | 1.12 | 1.13 | 1.14 | 1.15 |
| :------------- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 兼容性         | x    | ？   | ？   | ？   | ？   | ？   | ✓    |

- ✕ 不支持的版本范围。
- ✓ 完全支持的版本范围。
- ? 由于Kubernetes API版本之间的重大更改，某些功能可能无法在仪表板中正常运行。

## 三、部署 Kubernetes Dashboard

> 注意：dashboard-v2.0.0-beta4是部署在"kubernetes-dashboard"空间,如果“kubernetes-dashboard”命名空间已经存在 Kubernetes-Dashboard 相关资源，请换成别的 Namespace。v1和v2的dashboard是可以共存的。

完整部署文件 Github 地址：https://github.com/my-dlq/blog-example/tree/master/kubernetes/kubernetes-dashboard2.0.0-beta4-deploy

### 1、Dashboard RBAC

#### 创建 Dashboard RBAC 部署文件

**k8s-dashboard-rbac.yaml**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
```

#### 部署 Dashboard RBAC

```bash
$ kubectl apply -f k8s-dashboard-rbac.yaml
```

### 2、创建 ConfigMap、Secret

#### 创建 Dashboard Config & Secret 部署文件

**k8s-dashboard-configmap-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kube-system
type: Opaque
data:
  csrf: ""
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kube-system
type: Opaque
---
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kube-system
```

#### 部署 Dashboard Config & Secret

```bash
$ kubectl apply -f k8s-dashboard-configmap-secret.yaml
```

### 3、kubernetes-dashboard

#### 创建 Dashboard Deploy 部署文件

**k8s-dashboard-deploy.yaml**

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31001
  selector:
    k8s-app: kubernetes-dashboard
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.0.0-beta4
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kube-system          #设置为当前namespace
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
```

#### 部署 Dashboard Deploy

```
$ kubectl apply -f k8s-dashboard-deploy.yaml
```

### 4、创建 kubernetes-metrics-scraper

#### 创建 Dashboard Metrics 部署文件

**k8s-dashboard-metrics.yaml**

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kube-system
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: kubernetes-metrics-scraper
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-metrics-scraper
  name: kubernetes-metrics-scraper
  namespace: kube-system
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: kubernetes-metrics-scraper
    spec:
      containers:
        - name: kubernetes-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.1
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
      serviceAccountName: kubernetes-dashboard
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
```

#### 部署 Dashboard Metrics

```bash
$ kubectl apply -f k8s-dashboard-metrics.yaml
```

### 5、创建访问的 ServiceAccount

创建一个绑定 admin 权限的 ServiceAccount，获取其 Token 用于访问看板。

#### 创建 Dashboard ServiceAccount 部署文件

**k8s-dashboard-token.yaml**

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

#### 部署访问的 ServiceAccount

```bash
$ kubectl apply -f k8s-dashboard-token.yaml
```

#### 获取 Token

```bash
$ kubectl describe secret/$(kubectl get secret -n kube-system |grep admin|awk '{print $1}') -n kube-system
```

**token：**

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1iNGo0aCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjkwMTQzMWYxLTVmNGItMTFlOS05Mjg3LTAwMGMyOWQ5ODY5NyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.iwE1UdhB78FgXZJh4ByyOZVNh7M1l2CmOOevihOrY9tl_Z5sf3i_04CA33xA2LAMg7WNVYPjGB7vszBlkQyDGw0H5kJzIfL1YnR0JeLQkNk3v9TLyRqKJA2n8pxmJQIJP1xq0OPRGOfcA_n_c5qESs9QFHejVc5vABim8VBGX-pefKoJVXgu3r4w8gr1ORn4l5-LtHdQjSz3Dys7HwZo71fX2aLQR5bOPurkFKXqymcUoBYpWVsf-0cyN7hLRO-x-Z1i-uVpdM8ClpYSHv49eoDJePrcWpRp-Ryq6SNpGhiqCjjifEQAVHbr36QSAx8I1aamqLcpA0Da2qnunw52JA

## 四、登录新版本 Dashboard 查看

​    本人的 Kubernetes 集群地址为”192.168.2.11”并且在 Service 中设置了 NodePort 端口为 31001 和类型为 NodePort 方式访问 Dashboard ，所以访问地址：[https://192.168.2.11:31001](https://192.168.2.11:31001/) 进入 Kubernetes Dashboard 页面，然后输入上一步中创建的 ServiceAccount 的 Token 进入 Dashboard，可以看到新的 Dashboard。

[![img](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1002.png)](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1002.png)

​    可以感受到的是，这个页面比以前访问速度更加快速（估计是加了缓存），增加了暗黑模式，和编译对象时候增加了 yaml 格式的查看，整体风格更加简洁，并且新增角色对象可以直接在页面进行编译了。

[![img](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1003.png)](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1003.png)

[![img](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1004.png)](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1004.png)

[![img](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1005.png)](https://mydlq-club.oss-cn-beijing.aliyuncs.com/images/kubernetes-dashboard-bate1-1005.png)

## 五、问题处理

kubernetes-dashboard这个Pod老是起不来，CrashLoopBackOff，通过describe pod查看

```bash
# kubectl describe pod -n kubernetes-dashboard kubernetes-dashboard-748896587b-znbsh 
Name:           kubernetes-dashboard-748896587b-znbsh
Namespace:      kubernetes-dashboard
Node:           k8s-node1/192.168.1.224
Start Time:     Sun, 24 Nov 2019 10:12:44 +0800
Labels:         k8s-app=kubernetes-dashboard
                pod-template-hash=748896587b
Annotations:    <none>
Status:         Running
IP:             10.48.0.29
Controlled By:  ReplicaSet/kubernetes-dashboard-748896587b
Containers:
  kubernetes-dashboard:
    Container ID:  docker://80147ca1e1c596f230e63496c11cdaedb3e34496865fee0e508ed6e1274576ac
    Image:         harbor.dukanghub.com/k8s/dashboard:v2.0.0-beta4
    Image ID:      docker-pullable://harbor.dukanghub.com/k8s/dashboard@sha256:1dc220ba89df386cf8696d53093fc324a12f918b82aefb8deca3974b0b179d04
    Port:          8443/TCP
    Host Port:     0/TCP
    Args:
      --default-cert-dir=/certs
      --tls-cert-file=/certs/dashboard.pem
      --tls-key-file=/certs/dashboard-key.pem
      --token-ttl=43200
      --namespace=kubernetes-dashboard
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Sun, 24 Nov 2019 10:14:23 +0800
      Finished:     Sun, 24 Nov 2019 10:14:23 +0800
    Ready:          False
    Restart Count:  4
    Liveness:       http-get https://:8443/ delay=30s timeout=30s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /certs from kubernetes-dashboard-certs (rw)
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kubernetes-dashboard-token-gqmvd (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kubernetes-dashboard-certs:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-certs
    Optional:    false
  tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kubernetes-dashboard-token-gqmvd:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-token-gqmvd
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                 From                Message
  ----     ------     ----                ----                -------
  Normal   Scheduled  2m                  default-scheduler   Successfully assigned kubernetes-dashboard/kubernetes-dashboard-748896587b-znbsh to k8s-node1
  Normal   Pulling    68s (x4 over 118s)  kubelet, k8s-node1  Pulling image "harbor.dukanghub.com/k8s/dashboard:v2.0.0-beta4"
  Normal   Pulled     68s (x4 over 118s)  kubelet, k8s-node1  Successfully pulled image "harbor.dukanghub.com/k8s/dashboard:v2.0.0-beta4"
  Normal   Created    68s (x4 over 118s)  kubelet, k8s-node1  Created container kubernetes-dashboard
  Normal   Started    67s (x4 over 118s)  kubelet, k8s-node1  Started container kubernetes-dashboard
  Warning  BackOff    65s (x9 over 115s)  kubelet, k8s-node1  Back-off restarting failed container
```

注意几个信息：

- 退出状态码为1，

- 容器参数如下：

  ```yaml
  Args:
        --default-cert-dir=/certs
        --tls-cert-file=/certs/dashboard.pem
        --tls-key-file=/certs/dashboard-key.pem
        --token-ttl=43200
        --namespace=kubernetes-dashboard
  ```

- 容器是起来后又Back-off了，竟然就起来又挂掉了。这种情况基本容器自身程序原因自己退出。所以可以看下容器日志

  ```bash
  # kubectl logs -f -n kubernetes-dashboard kubernetes-dashboard-748896587b-znbsh 
  2019/11/24 02:15:46 Starting overwatch
  2019/11/24 02:15:46 Using namespace: kubernetes-dashboard
  2019/11/24 02:15:46 Using in-cluster config to connect to apiserver
  2019/11/24 02:15:46 Using secret token for csrf signing
  2019/11/24 02:15:46 Initializing csrf token from kubernetes-dashboard-csrf secret
  2019/11/24 02:15:46 Successful initial request to the apiserver, version: v1.15.0
  2019/11/24 02:15:46 Generating JWE encryption key
  2019/11/24 02:15:46 New synchronizer has been registered: kubernetes-dashboard-key-holder-kubernetes-dashboard. Starting
  2019/11/24 02:15:46 Starting secret synchronizer for kubernetes-dashboard-key-holder in namespace kubernetes-dashboard
  2019/11/24 02:15:46 Initializing JWE encryption key from synchronized object
  2019/11/24 02:15:46 Creating in-cluster Sidecar client
  2019/11/24 02:15:46 Error while loading dashboard server certificates. Reason: open /certs//certs/dashboard.pem: no such file or directory
  ```

  错误原因就是：`Error while loading dashboard server certificates. Reason: open /certs//certs/dashboard.pem: no such file or directory`

  这时对照我们上面的args时，就容易发现，当指定了`--default-cert-dir后面的证书只需指明文件名即可。于是改参数为如下：

  ```yaml
  Args:
        --default-cert-dir=/certs
        --tls-cert-file=dashboard.pem
        --tls-key-file=dashboard-key.pem
        --token-ttl=43200
        --namespace=kubernetes-dashboard
  ```

  重新apply后，即成功了

  ```bash
  # kubectl get pods -n kubernetes-dashboard 
  NAME                                        READY   STATUS    RESTARTS   AGE
  dashboard-metrics-scraper-d7bfdb8b7-f7ptk   1/1     Running   0          8m25s
  kubernetes-dashboard-5cdb544586-v66f9       1/1     Running   0          69s
  ```

  

