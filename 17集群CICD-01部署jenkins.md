[toc]

# 部署jenkins到k8s集群

## 1. 部署说明

- 镜像说明：目前jenkins官方在使用的为`jenkins/jenkins`和`jenkins/jnlp-slave`，`jenkinsci/jenkins`已在2018年12月初停止更新。`jenkins`这个仓库也废弃好久了，理由是：“ 我们停止使用和更新该镜像的简短原因是，我们每次发版时都需要人工参与。 ”
- jenkins官方推荐512MB内存以上，10GB存储空间

## 2. 部署jenkins

创建名称空间

```bash
kubectl create namespace jenkins
```

挂载时区文件

```bash
kubectl apply -f /apps/work/k8s/allow-lxcfs-tz-env.yaml -n jenkins
```

创建`jenkins-pvc.yaml`

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins
  namespace: jenkins
spec:
	storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi  #容量自己按需修改
```

创建`jenkins-rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins
  namespace: jenkins
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
```

后面jenkins发布应用都是使用这个service account发布，所以这个得改成`ClusterRole`和`ClusterRoleBinding`，否则应用只能发布到`jenkins`空间，不能发到其他空间。

创建`jenkins-deployment.yaml`

```yaml
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: harbor.dukanghub.com/jenkins/jenkins:lts
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000  #这个是与slave通信的端口
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 2000m
            memory: 4Gi
          requests:
            cpu: 1000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins
          mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent
```

如果要部署到指定的node上，请使用nodeSelector，如果node上有污点，还需要加上容忍，如下所示：

```yaml
nodeSelector:
  ingress-nginx: "yes"
tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/ingress
  operator: Equal
```

创建`jenkins-ingress.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web-ui
  namespace: jenkins #这个必须和下面的service所处空间一致
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - host: jenkins.dukanghub.com
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
```

**注意：**ingress规则里的`namespace`必须要下面`backend`里的`service`所处`namespace`一致。

接下面将此域名解析到`ingress-controller`所在服务器IP或VIP或外网IP，即可使用此域名访问`jenkins-ui`了。

如果需要证书，可将证书放于前端负载，反向代理到ingress。像上面我们是解析到ingress的，所以我们直接将证书放置到ingress规则里，需要先创建证书secret到jenkins空间。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web-ui
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.ingress.kubernetes.io/frontend-entry-points: http,https
spec:
  tls:
   - secretName: dukanghub-tls-cert
  rules:
  - host: jenkins.dukanghub.com
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
```

上面的规则，支持http,https访问

