[toc]

# 使用log-pilot收集日志

## 1. 软件介绍

- log-pilot是阿里云开源的一个日志收集工具
- github项目地址：https://github.com/AliyunContainerService/log-pilot
- 官方介绍：https://yq.aliyun.com/articles/674327
- 官方搭建：https://yq.aliyun.com/articles/674361?spm=a2c4e.11153940.0.0.21ae21c3mTKwWS

## 2. 准备工作

- 你有一个k8s集群

- 你有一个允许自动创建索引的elasticsearch集群

  默认情况下阿里云ElasticSearch集群当新增的文档发现没有索引时，不允许自动创建索引，而Log-Pilot在自动采集容器日志时需要依据日志采集配置来自动创建文档索引，因此我们这里需要开启自动创建索引功能：

  ![image](https://yqfile.alicdn.com/e3d58f26521b4c435856145c18cffde72fac2ffa.png)

  

## 3. 使用`DaemonSet`方式部署`log-pilot`

`vim log-pilot-DaemonSet.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: log-pilot
  labels:
    app: log-pilot
  # 设置期望部署的namespace
  namespace: kube-system
spec:
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: log-pilot
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      # 是否允许部署到Master节点上
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: log-pilot
        # 版本请参考https://github.com/AliyunContainerService/log-pilot/releases
        image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        env:
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: "LOGGING_OUTPUT"
            value: "elasticsearch"
          # 请确保集群到ES网络可达
          - name: "ELASTICSEARCH_HOSTS"
            value: "10.10.5.78:9200"
          # 配置ES访问权限
          #- name: "ELASTICSEARCH_USER"
          #  value: "{es_username}"
          #- name: "ELASTICSEARCH_PASSWORD"
          #  value: "{es_password}"
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: root
          mountPath: /host
          readOnly: true
        - name: varlib
          mountPath: /var/lib/filebeat
        - name: varlog
          mountPath: /var/log/filebeat
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        livenessProbe:
          failureThreshold: 3
          exec:
            command:
            - /pilot/healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: root
        hostPath:
          path: /
      - name: varlib
        hostPath:
          path: /var/lib/filebeat
          type: DirectoryOrCreate
      - name: varlog
        hostPath:
          path: /var/log/filebeat
          type: DirectoryOrCreate
      - name: localtime
        hostPath:
          path: /etc/localtime
```

参数说明：

1. {es_endpoint}：ES集群的访问地址，若跟Kubernetes集群在同一个VPC下，则可直接使用私网地址；`请务必确保到ES集群网络可达`。
2. {es_port}：ES集群的访问端口，一般是9200。
3. {es_username}：访问ES集群的用户名。
4. {es_password}：访问ES集群的用户密码。

**注意：**

- 时区的挂载并不是必需的，因为我们前面在每个空间创建了`allow-tz-env`这个`podpreset`资源

  ```bash
  # kubectl get podpresets -A
  NAMESPACE              NAME                 CREATED AT
  clusterstorage         allow-lxcfs-tz-env   2019-11-12T03:58:03Z
  default                allow-lxcfs-tz-env   2019-11-12T03:57:36Z
  kube-system            allow-lxcfs-tz-env   2019-11-04T03:16:13Z
  kubernetes-dashboard   allow-lxcfs-tz-env   2019-11-24T02:41:21Z
  monitoring             allow-lxcfs-tz-env   2019-11-12T03:57:56Z
  ```

- 对于有污点的node，需要增加容忍

## 4. 创建nginx测试pod收集日志示例

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: node-affinity
spec:
  selector:
    matchLabels:
      app: node-affinity
  replicas: 3
  template:
    metadata:
      labels:
        app: node-affinity
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        env:
        - name: aliyun_logs_nginx
          value: "stdout"
---
apiVersion: v1
kind: Service
metadata:
  name: node-affinity
spec:
  selector:
    app: node-affinity
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
```

## 5. 创建tomcat测试pod收集日志示例

 这里以一个Tomcat为例来说明如何配置应用的日志采集配置（配置方式同样适用Deploment和StatefulSet）： 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tomcat
spec:
  containers:
  - name: tomcat
    image: "tomcat:8.0"
    env:
    # 1、stdout为约定关键字，表示采集标准输出日志
    # 2、配置标准输出日志采集到ES的catalina索引下
    - name: aliyun_logs_catalina
      value: "stdout"
    # 1、配置采集容器内文件日志，支持通配符
    # 2、配置该日志采集到ES的access索引下
    - name: aliyun_logs_access
      value: "/usr/local/tomcat/logs/catalina.*.log"
    # 容器内文件日志路径需要配置emptyDir
    volumeMounts:
      - name: tomcat-log
        mountPath: /usr/local/tomcat/logs
  volumes:
    - name: tomcat-log
      emptyDir: {}
```

部署成功后稍等几秒钟，我们可以看到tomcat的日志被采集到了指定的ES集群中：
![image](https://yqfile.alicdn.com/827200d8372b492b5305893d1ca145a2cf0986da.png)

```
注意：Log-Pilot默认采集日志到ES集群时会自动创建格式为 index-yyyy.MM.dd 的索引
```

因此，我们可以看到只需要通过上面简单的配置就可以很方便地将Kubernetes集群中Pod的标准输出日志和容器内文件日志采集到ElasticSearch集群中。