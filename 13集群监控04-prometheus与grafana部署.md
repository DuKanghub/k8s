[toc]

## prometheus与grafana部署

https://blog.51cto.com/juestnow/2411335

### 1. 部署准备

```
说明：所有的容器组都运行在monitoring 命名空间
本文参考https://github.com/coreos/kube-prometheus
由于官方维护的版本在现有的部署环境出现问题所以下面做了一些修改及变更不影响整体效果
prometheus yaml 会做一些小的更改，grafana 不使用官方yaml 
```

### 2. 下载官方yaml

```bash
cd /apps/work/k8s
git clone https://github.com/coreos/kube-prometheus
cd kube-prometheus/manifests
# 虽然可以直接apply整个目录，把所有组件都一次性装上，但不利于我们理解。所以还是创建目录，分开放
mkdir prometheus-prometheus
# prometheus服务端有以下10个yaml文件
mv  prometheus-clusterRoleBinding.yaml                 prometheus-prometheus/
mv  prometheus-clusterRole.yaml                        prometheus-prometheus/
mv  prometheus-prometheus.yaml                         prometheus-prometheus/
mv  prometheus-roleBindingConfig.yaml                  prometheus-prometheus/
mv  prometheus-roleBindingSpecificNamespaces.yaml      prometheus-prometheus/
mv  prometheus-roleConfig.yaml                         prometheus-prometheus/
mv  prometheus-roleSpecificNamespaces.yaml             prometheus-prometheus/
mv  prometheus-rules.yaml                              prometheus-prometheus/
mv  prometheus-serviceAccount.yaml                     prometheus-prometheus/
mv  prometheus-service.yaml                            prometheus-prometheus/
# 创建监控yaml目录，以后所有系统业务的监控yaml 放到此文件夹
mkdir prometheus-monitor 
# 此项目自带以下监控,共7个文件
mv prometheus-serviceMonitor.yaml                        prometheus-monitor/
mv prometheus-serviceMonitorApiserver.yaml               prometheus-monitor/
mv prometheus-serviceMonitorCoreDNS.yaml                 prometheus-monitor/
mv prometheus-serviceMonitorKubeControllerManager.yaml   prometheus-monitor/
mv prometheus-serviceMonitorKubelet.yaml                 prometheus-monitor/
mv prometheus-serviceMonitorKubeScheduler.yaml           prometheus-monitor/
mv prometheus-operator-serviceMonitor.yaml               prometheus-monitor/
# 还有以下几个监控，放在其自身组件目录下吧
alertmanager-serviceMonitor.yaml
grafana-serviceMonitor.yaml
kube-state-metrics-serviceMonitor.yaml
node-exporter-serviceMonitor.yaml
```

**先搞明白这些yaml文件的大致作用:**

- prometheus服务端yaml文件

  | 文件名                                        | 作用                                                         | 备注                                                         |
  | --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | prometheus-clusterRoleBinding.yaml            | 将sa prometheus-k8s绑到prometheus-k8s这个ClusterRole上       |                                                              |
  | prometheus-clusterRole.yaml                   | 定义prometheus-k8s这个ClusterRole拥有的权限                  |                                                              |
  | prometheus-prometheus.yaml                    | 这是部署文件,需要修改镜像地址和增加数据持久化                | 要求内存400M，部署方式是StatefulSet                          |
  | prometheus-roleBindingConfig.yaml             | 将prometheus-k8s-config这个sa绑到prometheus-k8s-config这个Role上 |                                                              |
  | prometheus-roleBindingSpecificNamespaces.yaml | 定义了一个RoleBinding列表，将prometheus-k8s这个sa绑定default,kube-system,monitoring这三个空间对应的Role上 |                                                              |
  | prometheus-roleConfig.yaml                    | 定义prometheus-k8s-config这个Role的权限，可以get monitoring空间下的所有configmap |                                                              |
  | prometheus-roleSpecificNamespaces.yaml        | 定义了一个Role列表，对应上面的RoleBinding列表                | 能在default,kube-system,monitoring空间下get,list,watch services,endpoints,pods |
  | prometheus-rules.yaml                         | 定义了各种指标的采集规则                                     |                                                              |
  | prometheus-serviceAccount.yaml                | 定义prometheus-k8s这个SA                                     |                                                              |
  | prometheus-service.yaml                       | 定义prometheus-k8s这个Service，对应上面的部署文件            | 对外提供9090端口                                             |

- prometheus监控yaml文件

  | 文件名                                              | 作用                      | 备注 |
  | --------------------------------------------------- | ------------------------- | ---- |
  | prometheus-serviceMonitor.yaml                      | 监控Service               |      |
  | prometheus-serviceMonitorApiserver.yaml             | 监控Apiserver             |      |
  | prometheus-serviceMonitorCoreDNS.yaml               | 监控coredns               |      |
  | prometheus-serviceMonitorKubeControllerManager.yaml | 监控KubeControllerManager |      |
  | prometheus-serviceMonitorKubelet.yaml               | 监控kubelet               |      |
  | prometheus-serviceMonitorKubeScheduler.yaml         | 监控KubeScheduler         |      |
  | prometheus-operator-serviceMonitor.yaml             | 监控prometheus-operator   |      |

- grafana相关yaml文件

  | 文件名                            | 作用 | 备注 |
  | --------------------------------- | ---- | ---- |
  | grafana-dashboardDatasources.yaml |      |      |
  | grafana-dashboardSources.yaml     |      |      |
  | grafana-serviceAccount.yaml       |      |      |
  | grafana-service.yaml              |      |      |
  | grafana-dashboardDefinitions.yaml |      |      |
  | grafana-deployment.yaml           |      |      |
  | grafana-serviceMonitor.yaml       |      |      |

### 3. 创建监控`etcd secret`

`etcd`监控要用到证书同时需要修改`prometheus-prometheus.yaml`,不如先创建修改好，以后就不用那么麻烦了
`cd /apps/work/k8s/cfssl/pki/etcd/`
监控etcd 生成配置etcd-certs

```bash
kubectl -n monitoring create secret generic etcd-certs \
--from-file=etcd-ca.pem \
--from-file=etcd_client-key.pem \
--from-file=etcd_client.pem
```

### 4. 修改prometheus-prometheus.yaml

```bash
cd /apps/work/k8s/kube-prometheus/manifests/prometheus-prometheus
vim prometheus-prometheus.yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
  baseImage: quay.io/prometheus/prometheus
  nodeSelector:
    beta.kubernetes.io/os: linux
  replicas: 2
  secrets:
    # 添加etcd 证书
  - etcd-certs
  resources:
    requests:
      # 设置内存大小
      memory: 2048Mi
    # 设置数据保留天数
  retention: 7d
  # 创建外部存储pvc
  storage:
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        metadata:
          annotations:
             storageclass.kubernetes.io/is-default-class: true # 使用那种动态存储 设置默认动态存储
          labels:
            prometheus: ingest-webrtc
        resources:
          requests:
            storage: 200Gi # 存储空间大小
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.10.0 # 版本号
```

### 5. 执行yaml 创建prometheus 服务

```bash
kubectl apply -f .
```

### 6. 查看prometheus 服务 状态

```bash
# kubectl get pod -n monitoring -o wide | grep prometheus-k8s
prometheus-k8s-0                      3/3     Running   1          6m48s   10.48.0.38   k8s-node1   <none>           <none>
prometheus-k8s-1                      3/3     Running   1          7m6s    10.48.1.20   k8s-node2   <none>           <none>
# kubectl get service -n monitoring -o wide | grep prometheus-k8s
prometheus-k8s          ClusterIP   10.64.193.142   <none>        9090/TCP                     7m38s   app=prometheus,prometheus=k8s
# kubectl get statefulset -n monitoring  | grep prometheus-k8s
prometheus-k8s      2/2     113m
```

Alerts 状态 prometheus查看
`http://SERVICE_IP:9090/alerts`

### 7. 创建k8s 常见监控

#### 7.1 kube-apiserver 监控创建

```bash
cd ../prometheus-monitor
vim  prometheus-serviceMonitorApiserver.yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: apiserver
  name: kube-apiserver
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 30s
    port: https
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      serverName: kubernetes
  jobLabel: component
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      component: apiserver
      provider: kubernetes
```

#### 7.2 kubelet 监控创建

`vim prometheus-serviceMonitorKubelet.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kubelet
  name: kubelet
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    interval: 30s
    port: https-metrics
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    honorLabels: true
    interval: 30s
    path: /metrics/cadvisor
    port: https-metrics
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kubelet
```

#### 7.3 etcd 监控创建

etcd 不在k8s集群内部所以要创建Endpoints
`vim prometheus-serviceMonitoretcd.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: etcd
  name: etcd
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: https-metrics
    port: 2379
    protocol: TCP
    targetPort: 2379
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: etcd
  name: etcd
  namespace: monitoring
subsets:
- addresses:
  - ip: 192.168.1.221
  - ip: 192.168.1.222
  - ip: 192.168.1.223
  ports:
  - name: https-metrics
    port: 2379
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: etcd
  name: etcd
  namespace: monitoring
spec:
  endpoints:
  - interval: 10s
    port: https-metrics
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/etcd-ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/etcd_client.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd_client-key.pem
      insecureSkipVerify: true
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: etcd
```

#### 7.4 kube-scheduler 监控

不在k8s集群内部所以要创建Endpoints 删除官方提供的yaml

**注意：**为啥前面的`apiserver`和`kubelet`无需指明`Endpoints`，因为这两个开发时已经指明了，可使用`kubectl get endpoints`查看

```bash
# kubectl get endpoints
NAME                      ENDPOINTS                                                                 AGE
kube-controller-manager   <none>                                                                    33d
kube-dns                  10.48.0.18:53,10.48.1.4:53,10.48.1.5:53 + 6 more...                       5d
kube-scheduler            <none>                                                                    33d
kubelet                   192.168.1.216:10255,192.168.1.218:10255,192.168.1.224:10255 + 9 more...   21h
metrics-server            10.48.0.23:4443                                                           3d21h
```

`vim prometheus-serviceMonitorKubeScheduler.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP
    targetPort: 10251
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
subsets:
- addresses:
  - ip: 192.168.1.221
  - ip: 192.168.1.222
  - ip: 192.168.1.223
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 10s
    port: http-metrics
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: kube-scheduler
```

#### 7.5 kube-controller-manager 监控

不在k8s集群内部所以要创建Endpoints 删除官方提供的yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: http-controller
    port: 10252
    protocol: TCP
    targetPort: 10252
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
subsets:
- addresses:
  - ip: 192.168.1.221
  - ip: 192.168.1.222
  - ip: 192.168.1.223
  ports:
  - name: http-controller
    port: 10252
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 10s
    port: http-controller
  namespaceSelector:
    matchNames:
    - monitoring
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
```

#### 7.6 CoreDNS 监控

`vim prometheus-serviceMonitorCoreDNS.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: coredns
  name: coredns
  namespace: monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 15s
    port: metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-dns
```

#### 7.8 执行yaml 创建ServiceMonitor

```bash
kubectl apply -f .
```

等待几分钟后打开`http://10.64.61.191:9090`(IP是prometheus的serviceIP),
`targets`标签是否有对应的服务如果有一切正常

![image-20191126144545675](imgs/image-20191126144545675.png)

如果哪个指标没有，可以手动查看，比如scheduler的指标，可以通过`http://nodeIP:10251/metrics`查看

### 8. 创建 grafana 服务相关的yaml文件不使用官网提供yaml

#### 8.1 创建grafana-pvc

```bash
mkdir grafana
cd grafana
vim grafana-pvc.yaml
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 50Gi
```

#### 8.2 grafana-configmap

`vim grafana-configmap.yaml`

```yaml
apiVersion: v1
data:
  grafana.ini: |
    ##################### Grafana Configuration Example #####################
    #
    # Everything has defaults so you only need to uncomment things you want to
    # change

    # possible values : production, development
    ; app_mode = production

    # instance name, defaults to HOSTNAME environment variable value or hostname if HOSTNAME var is empty
    ; instance_name = ${HOSTNAME}

    #################################### Paths ####################################
    [paths]
    # Path to where grafana can store temp files, sessions, and the sqlite3 db (if that is used)
    ;data = /var/lib/grafana

    # Directory where grafana can store logs
    ;logs = /var/log/grafana

    # Directory where grafana will automatically scan and look for plugins
    ;plugins = /var/lib/grafana/plugins

    # folder that contains provisioning config files that grafana will apply on startup and while running.
    ; provisioning = conf/provisioning

    #################################### Server ####################################
    [server]
    # Protocol (http, https, socket)
    ;protocol = http

    # The ip address to bind to, empty will bind to all interfaces
    ;http_addr =

    # The http port  to use
    ;http_port = 3000

    # The public facing domain name used to access grafana from a browser
    ;domain = localhost

    # Redirect to correct domain if host header does not match domain
    # Prevents DNS rebinding attacks
    ;enforce_domain = false

    # The full public facing url you use in browser, used for redirects and emails
    # If you use reverse proxy and sub path specify full url (with sub path)
    ;root_url = http://localhost:3000

    # Log web requests
    ;router_logging = false

    # the path relative working path
    ;static_root_path = public

    # enable gzip
    ;enable_gzip = false

    # https certs & key file
    ;cert_file =
    ;cert_key =

    # Unix socket path
    ;socket =

    #################################### Database ####################################
    [database]
    # You can configure the database connection by specifying type, host, name, user and password
    # as seperate properties or as on string using the url propertie.

    # Either "mysql", "postgres" or "sqlite3", it's your choice
    ;type = sqlite3
    ;host = 127.0.0.1:3306
    ;name = grafana
    ;user = root
    # If the password contains # or ; you have to wrap it with triple quotes. Ex """#password;"""
    ;password =

    # Use either URL or the previous fields to configure the database
    # Example: mysql://user:secret@host:port/database
    ;url =

    # For "postgres" only, either "disable", "require" or "verify-full"
    ;ssl_mode = disable

    # For "sqlite3" only, path relative to data_path setting
    ;path = grafana.db

    # Max idle conn setting default is 2
    ;max_idle_conn = 2

    # Max conn setting default is 0 (mean not set)
    ;max_open_conn =

    # Connection Max Lifetime default is 14400 (means 14400 seconds or 4 hours)
    ;conn_max_lifetime = 14400

    # Set to true to log the sql calls and execution times.
    log_queries =

    #################################### Session ####################################
    [session]
    # Either "memory", "file", "redis", "mysql", "postgres", default is "file"
    ;provider = file

    # Provider config options
    # memory: not have any config yet
    # file: session dir path, is relative to grafana data_path
    # redis: config like redis server e.g. `addr=127.0.0.1:6379,pool_size=100,db=grafana`
    # mysql: go-sql-driver/mysql dsn config string, e.g. `user:password@tcp(127.0.0.1:3306)/database_name`
    # postgres: user=a password=b host=localhost port=5432 dbname=c sslmode=disable
    ;provider_config = sessions

    # Session cookie name
    ;cookie_name = grafana_sess

    # If you use session in https only, default is false
    ;cookie_secure = false

    # Session life time, default is 86400
    ;session_life_time = 86400

    #################################### Data proxy ###########################
    [dataproxy]

    # This enables data proxy logging, default is false
    ;logging = false

    #################################### Analytics ####################################
    [analytics]
    # Server reporting, sends usage counters to stats.grafana.org every 24 hours.
    # No ip addresses are being tracked, only simple counters to track
    # running instances, dashboard and error counts. It is very helpful to us.
    # Change this option to false to disable reporting.
    ;reporting_enabled = true

    # Set to false to disable all checks to https://grafana.net
    # for new vesions (grafana itself and plugins), check is used
    # in some UI views to notify that grafana or plugin update exists
    # This option does not cause any auto updates, nor send any information
    # only a GET request to http://grafana.com to get latest versions
    ;check_for_updates = true

    # Google Analytics universal tracking code, only enabled if you specify an id here
    ;google_analytics_ua_id =

    #################################### Security ####################################
    [security]
    # default admin user, created on startup
    ;admin_user = admin

    # default admin password, can be changed before first start of grafana,  or in profile settings
    ;admin_password = admin

    # used for signing
    ;secret_key = SW2YcwTIb9zpOOhoPsMm

    # Auto-login remember days
    ;login_remember_days = 7
    ;cookie_username = grafana_user
    ;cookie_remember_name = grafana_remember

    # disable gravatar profile images
    ;disable_gravatar = false

    # data source proxy whitelist (ip_or_domain:port separated by spaces)
    ;data_source_proxy_whitelist =

    # disable protection against brute force login attempts
    ;disable_brute_force_login_protection = false

    #################################### Snapshots ###########################
    [snapshots]
    # snapshot sharing options
    ;external_enabled = true
    ;external_snapshot_url = https://snapshots-origin.raintank.io
    ;external_snapshot_name = Publish to snapshot.raintank.io

    # remove expired snapshot
    ;snapshot_remove_expired = true

    #################################### Dashboards History ##################
    [dashboards]
    # Number dashboard versions to keep (per dashboard). Default: 20, Minimum: 1
    ;versions_to_keep = 20

    #################################### Users ###############################
    [users]
    # disable user signup / registration
    ;allow_sign_up = true

    # Allow non admin users to create organizations
    ;allow_org_create = true

    # Set to true to automatically assign new users to the default organization (id 1)
    ;auto_assign_org = true

    # Default role new users will be automatically assigned (if disabled above is set to true)
    ;auto_assign_org_role = Viewer

    # Background text for the user field on the login page
    ;login_hint = email or username

    # Default UI theme ("dark" or "light")
    ;default_theme = dark

    # External user management, these options affect the organization users view
    ;external_manage_link_url =
    ;external_manage_link_name =
    ;external_manage_info =

    # Viewers can edit/inspect dashboard settings in the browser. But not save the dashboard.
    ;viewers_can_edit = false

    [auth]
    # Set to true to disable (hide) the login form, useful if you use OAuth, defaults to false
    ;disable_login_form = false

    # Set to true to disable the signout link in the side menu. useful if you use auth.proxy, defaults to false
    ;disable_signout_menu = false

    #################################### Anonymous Auth ##########################
    [auth.anonymous]
    # enable anonymous access
    ;enabled = false

    # specify organization name that should be used for unauthenticated users
    ;org_name = Main Org.

    # specify role for unauthenticated users
    ;org_role = Viewer

    #################################### Github Auth ##########################
    [auth.github]
    ;enabled = false
    ;allow_sign_up = true
    ;client_id = some_id
    ;client_secret = some_secret
    ;scopes = user:email,read:org
    ;auth_url = https://github.com/login/oauth/authorize
    ;token_url = https://github.com/login/oauth/access_token
    ;api_url = https://api.github.com/user
    ;team_ids =
    ;allowed_organizations =

    #################################### Google Auth ##########################
    [auth.google]
    ;enabled = false
    ;allow_sign_up = true
    ;client_id = some_client_id
    ;client_secret = some_client_secret
    ;scopes = https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email
    ;auth_url = https://accounts.google.com/o/oauth2/auth
    ;token_url = https://accounts.google.com/o/oauth2/token
    ;api_url = https://www.googleapis.com/oauth2/v1/userinfo
    ;allowed_domains =

    #################################### Generic OAuth ##########################
    [auth.generic_oauth]
    ;enabled = false
    ;name = OAuth
    ;allow_sign_up = true
    ;client_id = some_id
    ;client_secret = some_secret
    ;scopes = user:email,read:org
    ;auth_url = https://foo.bar/login/oauth/authorize
    ;token_url = https://foo.bar/login/oauth/access_token
    ;api_url = https://foo.bar/user
    ;team_ids =
    ;allowed_organizations =

    #################################### Grafana.com Auth ####################
    [auth.grafana_com]
    ;enabled = false
    ;allow_sign_up = true
    ;client_id = some_id
    ;client_secret = some_secret
    ;scopes = user:email
    ;allowed_organizations =

    #################################### Auth Proxy ##########################
    [auth.proxy]
    ;enabled = false
    ;header_name = X-WEBAUTH-USER
    ;header_property = username
    ;auto_sign_up = true
    ;ldap_sync_ttl = 60
    ;whitelist = 192.168.1.1, 192.168.2.1

    #################################### Basic Auth ##########################
    [auth.basic]
    ;enabled = true

    #################################### Auth LDAP ##########################
    [auth.ldap]
    ;enabled = false
    ;config_file = /etc/grafana/ldap.toml
    ;allow_sign_up = true

    #################################### SMTP / Emailing ##########################
    [smtp]
    ;enabled = false
    ;host = localhost:25
    ;user =
    # If the password contains # or ; you have to wrap it with trippel quotes. Ex """#password;"""
    ;password =
    ;cert_file =
    ;key_file =
    ;skip_verify = false
    ;from_address = admin@grafana.localhost
    ;from_name = Grafana
    # EHLO identity in SMTP dialog (defaults to instance_name)
    ;ehlo_identity = dashboard.example.com

    [emails]
    ;welcome_email_on_sign_up = false

    #################################### Logging ##########################
    [log]
    # Either "console", "file", "syslog". Default is console and  file
    # Use space to separate multiple modes, e.g. "console file"
    ;mode = console file

    # Either "debug", "info", "warn", "error", "critical", default is "info"
    ;level = info

    # optional settings to set different levels for specific loggers. Ex filters = sqlstore:debug
    ;filters =

    # For "console" mode only
    [log.console]
    ;level =

    # log line format, valid options are text, console and json
    ;format = console

    # For "file" mode only
    [log.file]
    ;level =

    # log line format, valid options are text, console and json
    ;format = text

    # This enables automated log rotate(switch of following options), default is true
    ;log_rotate = true

    # Max line number of single file, default is 1000000
    ;max_lines = 1000000

    # Max size shift of single file, default is 28 means 1 << 28, 256MB
    ;max_size_shift = 28

    # Segment log daily, default is true
    ;daily_rotate = true

    # Expired days of log file(delete after max days), default is 7
    ;max_days = 7

    [log.syslog]
    ;level =

    # log line format, valid options are text, console and json
    ;format = text

    # Syslog network type and address. This can be udp, tcp, or unix. If left blank, the default unix endpoints will be used.
    ;network =
    ;address =

    # Syslog facility. user, daemon and local0 through local7 are valid.
    ;facility =

    # Syslog tag. By default, the process' argv[0] is used.
    ;tag =

    #################################### Alerting ############################
    [alerting]
    # Disable alerting engine & UI features
    ;enabled = true
    # Makes it possible to turn off alert rule execution but alerting UI is visible
    ;execute_alerts = true

    #################################### Internal Grafana Metrics ##########################
    # Metrics available at HTTP API Url /metrics
    [metrics]
    # Disable / Enable internal metrics
    ;enabled           = true

    # Publish interval
    ;interval_seconds  = 10

    # Send internal metrics to Graphite
    [metrics.graphite]
    # Enable by setting the address setting (ex localhost:2003)
    ;address =
    ;prefix = prod.grafana.%(instance_name)s.

    #################################### Distributed tracing ############
    [tracing.jaeger]
    # Enable by setting the address sending traces to jaeger (ex localhost:6831)
    ;address = localhost:6831
    # Tag that will always be included in when creating new spans. ex (tag1:value1,tag2:value2)
    ;always_included_tag = tag1:value1
    # Type specifies the type of the sampler: const, probabilistic, rateLimiting, or remote
    ;sampler_type = const
    # jaeger samplerconfig param
    # for "const" sampler, 0 or 1 for always false/true respectively
    # for "probabilistic" sampler, a probability between 0 and 1
    # for "rateLimiting" sampler, the number of spans per second
    # for "remote" sampler, param is the same as for "probabilistic"
    # and indicates the initial sampling rate before the actual one
    # is received from the mothership
    ;sampler_param = 1

    #################################### Grafana.com integration  ##########################
    # Url used to to import dashboards directly from Grafana.com
    [grafana_com]
    ;url = https://grafana.com

    #################################### External image storage ##########################
    [external_image_storage]
    # Used for uploading images to public servers so they can be included in slack/email messages.
    # you can choose between (s3, webdav, gcs, azure_blob, local)
    ;provider =

    [external_image_storage.s3]
    ;bucket =
    ;region =
    ;path =
    ;access_key =
    ;secret_key =

    [external_image_storage.webdav]
    ;url =
    ;public_url =
    ;username =
    ;password =

    [external_image_storage.gcs]
    ;key_file =
    ;bucket =
    ;path =

    [external_image_storage.azure_blob]
    ;account_name =
    ;account_key =
    ;container_name =

    [external_image_storage.local]
    # does not require any configuration

kind: ConfigMap
metadata:
  creationTimestamp: null
  name: grafana-etc
  namespace: monitoring
```

#### 8.3 grafana-deployment

`vim grafana-deployment.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana-core
  namespace: monitoring
  labels:
    app: grafana
    component: core
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
        component: core
    spec:
      containers:
      - image: grafana/grafana
        name: grafana-core
        imagePullPolicy: IfNotPresent
        # env:
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 200m
            memory: 2048Mi
          requests:
            cpu: 200m
            memory: 2048Mi
        env:
          # The following env variables set up basic auth twith the default admin user and admin password.
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          #   value: Admin
          # does not really work, because of template variables in exported dashboards:
          # - name: GF_DASHBOARDS_JSON_ENABLED
          #   value: "true"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
          # initialDelaySeconds: 30
          # timeoutSeconds: 1
        volumeMounts:
        - name: grafana-etc-volume
          mountPath: /etc/grafana/
        - name: grafana-settings
          mountPath: /var/lib/grafana
      volumes:
        - name: grafana-etc-volume
          configMap:
            name: grafana-etc
        - name: grafana-settings
          persistentVolumeClaim:
            claimName: grafana-pvc
```

#### 8.4 grafana-service

`vi grafana-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
    component: core
  annotations:
    prometheus.io/scrape: 'true'
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30091
      protocol: TCP
      name: webui
  selector:
    app: grafana
    component: core
```

#### 8.5 grafana-Monitor

`vim grafana-Monitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: grafana
  namespace: monitoring
spec:
  endpoints:
  - interval: 15s
    port: webui
  selector:
    matchLabels:
      app: grafana
```

### 9 执行yaml创建grafana服务

```bash
kubectl apply -f .
```

### 10 查看grafana状态

```bash
kubectl get service -n monitoring -o wide | grep grafana
kubectl get pod -n monitoring -o wide | grep grafana 
```

### 11 打开grafana

```bash
PodIP:3000或ServiceIP:3000
```

配置grafana数据源

### 12 导入grafana dashboards 文件

```bash
mkdir dashboards
cd dashboards
# 下载json 文件到此文件夹
# 我收集的一些 
https://github.com/qist/kubernetes-monitor/tree/master/prometheus/grafana/dashboards
kubectl create configmap "grafana-import-dashboards" \
--from-file=dashboards/ --namespace=monitoring
```

创建导入`dashboards job`
`vim grafana-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: grafana-import-dashboards
  namespace: monitoring
  labels:
    app: grafana
    component: import-dashboards
spec:
  template:
    metadata:
      name: grafana-import-dashboards
      labels:
        app: grafana
        component: import-dashboards
      annotations:
        pod.beta.kubernetes.io/init-containers: '[
          {
            "name": "wait-for-endpoints",
            "image": "registry.cn-hangzhou.aliyuncs.com/nevermore/grafana-tiny-tools:v1",
            "imagePullPolicy": "IfNotPresent",
            "command": ["fish", "-c", "echo \"waiting for endpoints...\"; while true; set endpoints (curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header \"Authorization: Bearer \"(cat /var/run/secrets/kubernetes.io/serviceaccount/token) https://kubernetes.default.svc/api/v1/namespaces/monitoring/endpoints/grafana); echo $endpoints | jq \".\"; if test (echo $endpoints | jq -r \".subsets[]?.addresses // [] | length\") -gt 0; exit 0; end; echo \"waiting...\";sleep 1; end"],
            "args": ["monitoring", "grafana"]
          }
        ]'
    spec:
      serviceAccountName: prometheus-k8s
      containers:
      - name: grafana-import-dashboards
        image: registry.cn-hangzhou.aliyuncs.com/nevermore/grafana-tiny-tools:v1
        command: ["/bin/sh", "-c"]
        workingDir: /opt/grafana-import-dashboards
        args:
          - >
            for file in *-datasource.json ; do
              if [ -e "$file" ] ; then
                echo "importing $file" &&
                curl --silent --fail --show-error \
                  --request POST http://admin:admin@grafana:3000/api/datasources \
                  --header "Content-Type: application/json" \
                  --data-binary "@$file" ;
                echo "" ;
              fi
            done ;
            for file in *-dashboard.json ; do
              if [ -e "$file" ] ; then
                echo "importing $file" &&
                ( echo '{"dashboard":'; \
                  cat "$file"; \
                  echo ',"overwrite":true,"inputs":[{"name":"DS_PROMETHEUS","type":"datasource","pluginId":"prometheus","value":"prometheus"}]}' ) \
                | jq -c '.' \
                | curl --silent --fail --show-error \
                  --request POST http://admin:admin@grafana:3000/api/dashboards/import \
                  --header "Content-Type: application/json" \
                  --data-binary "@-" ;
                echo "" ;
              fi
            done
        volumeMounts:
        - name: config-volume
          mountPath: /opt/grafana-import-dashboards
      restartPolicy: Never
      volumes:
      - name: config-volume
        configMap:
          name: grafana-import-dashboards
```

创建job，导入dashboard等数据

```bash
kubectl apply -f grafana-job.yaml
```

等待job 执行完成
打开grafana 
查看Prometheus 数据库是否配置成功
界面是否有dashboards 
![](imgs/2019-10-24-16-37-43.png)
如果有这样的数据证明就上传成功
Kubernetes 有些展示可能还有一些问题 state-metrics 还没 部署完成下一篇部署完成展示就会正常