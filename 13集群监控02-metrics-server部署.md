[toc]

## 13. metrics-server部署

说明：所有的容器组都运行在kube-system 命名空间

**注意：**由于是在master上使用`kubectl top`命令，master上一定要能和`Pod`和`Service`网段直接通信，可在master上添加路由表或在节点上级路由添加路由表。

本文参考: `https://github.com/kubernetes-incubator/metrics-server`
`https://blog.51cto.com/juestnow/2414189`

```bash
# 创建metrics-server  运行的label
kubectl label nodes  k8s-node1  metrics=yes
kubectl label nodes  k8s-node2  metrics=yes
```

### 13.1 准备yaml文件

```bash
cd /apps/work/k8s/kubernetes/cluster/addons/metrics-server
```

目录里有这里文件

```
.
├── auth-delegator.yaml
├── auth-reader.yaml
├── metrics-apiservice.yaml
├── metrics-server-deployment.yaml
├── metrics-server-service.yaml
├── OWNERS
├── README.md
└── resource-reader.yaml

0 directories, 8 files
```

`vim aggregated-metrics-reader.yaml`这个文件可以不需要

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:aggregated-metrics-reader
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

`vim auth-delegator.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
```

`vim auth-reader.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
```

`vim metrics-apiservice.yaml`

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

`vim metrics-server-deployment.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      tolerations:
        - effect: NoSchedule
          key: node.kubernetes.io/unschedulable
          operator: Exists
        - key: NoSchedule
          operator: Exists
          effect: NoSchedule
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: juestnow/metrics-server-amd64:v0.3.3
        imagePullPolicy: Always
        command:
        - /metrics-server  #这个参数看版本，0.3.6镜像自带这个
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        - --kubelet-insecure-tls # kubelet有server证书的可无需指定此参数
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
      nodeSelector:
        metrics: "yes"
```

`vim metrics-server-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/name: "Metrics-server"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - port: 443
    protocol: TCP
    targetPort: 443
```

`vim resource-reader.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
```

### 13.2 创建metrics-server 证书

```bash
cat << EOF | tee /apps/work/k8s/cfssl/k8s/metrics-server.json
{
  "CN": "metrics-server",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "GuangDong",
      "L": "GuangZhou",
      "O": "dukang",
      "OU": "dukang"
    }
  ]
}
EOF
```

生成证书

```bash
cfssl gencert \
-ca=/apps/work/k8s/cfssl/pki/k8s/k8s-ca.pem \
-ca-key=/apps/work/k8s/cfssl/pki/k8s/k8s-ca-key.pem \
-config=/apps/work/k8s/cfssl/ca-config.json \
-profile=kubernetes /apps/work/k8s/cfssl/k8s/metrics-server.json | \
cfssljson -bare ./metrics-server
```

创建secret

```bash
kubectl -n kube-system \
create secret generic metrics-server-certs \
--from-file=metrics-server-key.pem \
--from-file=metrics-server.pem
```

查看secret

```bash
kubectl get secret -n kube-system | grep metrics-server-certs
kubectl get secret metrics-server-certs -n kube-system  -o yaml
```

### 13.3 修改`metrics-server-deployment.yaml`

`vim metrics-server-deployment.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      tolerations:
        - effect: NoSchedule
          key: node.kubernetes.io/unschedulable
          operator: Exists
        - key: NoSchedule
          operator: Exists
          effect: NoSchedule
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      - name: metrics-server-certs
        secret:
          secretName: metrics-server-certs
      containers:
      - name: metrics-server
        image: juestnow/metrics-server-amd64:v0.3.3
        imagePullPolicy: Always
        command:
        - /metrics-server
        - --tls-cert-file=/certs/metrics-server.pem
        - --tls-private-key-file=/certs/metrics-server-key.pem
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        #- --kubelet-insecure-tls  ## kubelet 只签发客户端证书时必须添加这个参数，签发server 证书时不不要添加此参数
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
        - name: metrics-server-certs
          mountPath: /certs
      nodeSelector:
        metrics: "yes"
```

### 13.4 执行yaml文件

```bash
kubectl apply -f .
```

### 13.5 查看metrics-server状态

```bash
kubectl get pod -A | grep metrics-server
kubectl get svc -A | grep metrics-server
kubectl top pods
kubectl top node
root@DESKTOP-Q73T5I0:/apps/work/k8s/metrics-server# kubectl top node
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-node1   152m         8%     809Mi           23%       
k8s-node2   64m          3%     776Mi           24%       
lb01        40m          2%     278Mi           8%        
lb02        33m          1%     242Mi           7%  
```

**注意：**由于是在master上使用`kubectl top`命令，master上一定要能和`Pod`和`Service`网段直接通信，可在master上添加路由表或在节点上级路由添加路由表。

```bash
# 10.48.0.0/12是pod网段,192.168.1.217是负载均衡的IP，如果没有负载均衡，可写任意一个nodeIP
ip route add 10.48.0.0/12 via 192.168.1.217
# 10.64.0.0/16是Service网段
ip route add 10.64.0.0/16 via 192.168.1.217
```

也可以在上级路由添加静态路由，我是在上级路由添加的静态路由