# kubernetes 二进制部署ansible playbook 一键生成

#### 脚本仓库地址

```
https://github.com/qist/k8s
#支持 Ubuntu 18及以上的系统，CentOS7及CentOS8 系统
# k8s 版本 14，15，16，17 号版本
```

# ansible 安装

```bash
#Ubuntu  系列安装
apt -y install ansible
#CentOS 8 安装
dnf -y install ansible
# CentOS 7 安装
yum -y install ansible
# 修改ansible 配置
## 说明id_rsa_storm1 私钥名字请自行修改
sed -i 's/^#private_key_file =.*$/private_key_file =\/root\/.ssh\/id_rsa_storm1/g' /etc/ansible/ansible.cfg
sed -i 's/^#sudo_user      = root/sudo_user      = root/g' /etc/ansible/ansible.cfg
sed -i 's/^#remote_port    = 22/remote_port    = 22/g' /etc/ansible/ansible.cfg
sed -i 's/^#host_key_checking = False/host_key_checking = False/g' /etc/ansible/ansible.cfg
sed -i '/\[ssh_connection\]/a\ssh_args = -o ControlMaster=no' /etc/ansible/ansible.cfg
```

# 下载 CFSSL 二进制文件/或者自行编译

```bash
# 配置CFSSL编译环境 一点要在跨系统部署请在低版本下编译会用到lib库高版本编译在低版本上运行要升级库
#Ubuntu  系列安装
apt -y install gcc git
#CentOS 8 安装
dnf -y install gcc git 
# CentOS 7 安装
yum -y install gcc git
# 配置go 编译环境
# 下载go 语言
wget -P /usr/local/src/ https://dl.google.com/go/go1.12.14.linux-amd64.tar.gz
# 解压 
tar -xf  /usr/local/src/go1.12.14.linux-amd64.tar.gz -C  /usr/local/
# 配置环境变量
cat << EOF |tee /etc/profile
export GOPATH=/root/go
export GOBIN=/root/go/bin
PATH=\$PATH:/usr/local/go/bin:\$HOME/bin:\$GOBIN
export PATH
EOF
# 生效环境变量
source /etc/profile
# 编译 CFSSL
go get  github.com/cloudflare/cfssl/cmd/cfssl
go get  github.com/cloudflare/cfssl/cmd/cfssljson
#查看cfssl 是否安装成功
cfssl  version 
# 已经编译好的二进制文件 
wget -P  /tmp/ https://github.com/qist/lxcfs/releases/download/cfssl/cfssl.tar.gz
# 解压下载好文件
 tar -xf /tmp/cfssl.tar.gz -C /usr/bin/
 # 删除下载的压缩包
 rm -rf  /tmp/cfssl.tar.gz
```

# 下载kubectl 可以是任意版本kubectl

```bash
wget -P  /tmp/ https://storage.googleapis.com/kubernetes-release/release/v1.14.10/kubernetes-client-linux-amd64.tar.gz
# 解压
tar -xf /tmp/kubernetes-client-linux-amd64.tar.gz -C /tmp/
# cp 文件到/usr/bin
mv /tmp/kubernetes/client/bin/kubectl /usr/bin/
# 验证是否能执行
kubectl version 
# 删除 没用文件
 rm -rf /tmp/kubernetes-client-linux-amd64.tar.gz
 rm -rf   /tmp/kubernetes
```

# 下载 ansible 一键生成脚本

```
cd /opt
git clone https://github.com/qist/k8s.git
```

# 以 kubernetes.v1.17 脚本为例

### 准备下载使用的压缩包文件脚本不做下载，github 外网速度越来越慢下载成功率不高所有采用手动下载方式进行下载

```bash
打开kubernetes.v1.17.sh 查看下载路径 版本可以自行选择
# 创建压缩包存放目录
mkdir /tmp/source
# 下载K8S 集群所需压缩包
#docker 下载
wget -P /tmp/source https://download.docker.com/linux/static/stable/x86_64/docker-19.03.5.tgz
#lxcfs 下载  
wget -P /tmp/source https://github.com/qist/lxcfs/releases/download/3.1.2/lxcfs-3.1.2.tar.gz
# cni 下载
wget -P /tmp/source https://github.com/containernetworking/plugins/releases/download/v0.8.3/cni-plugins-linux-amd64-v0.8.3.tgz
# etcd 下载
wget -P /tmp/source https://github.com/etcd-io/etcd/releases/download/v3.3.18/etcd-v3.3.18-linux-amd64.tar.gz
# 下载kubernetes server 压缩包
wget -P /tmp/source https://storage.googleapis.com/kubernetes-release/release/v1.17.0/kubernetes-server-linux-amd64.tar.gz
# 下载haproxy 
wget -P /tmp/source https://www.haproxy.org/download/2.1/src/haproxy-2.1.1.tar.gz
# automake keepalived编译用到
wget -P /tmp/source https://ftp.gnu.org/gnu/automake/automake-1.15.1.tar.gz
# 下载keepalived
wget -P /tmp/source https://www.keepalived.org/software/keepalived-2.0.19.tar.gz
# iptables centos7 及Ubuntu18号版本用到
wget -P /tmp/source https://www.netfilter.org/projects/iptables/files/iptables-1.6.2.tar.bz2
```

# 修改kubernetes.v1.17.sh 文件改成自己环境所需配置信息

```bash
# 应用部署目录 可根据自己环境修改
TOTAL_PATH=/apps
ETCD_PATH=$TOTAL_PATH/etcd
# 大规模集群部署时建议分开存储，WAL 最好ssd
ETCD_DATA_DIR=$TOTAL_PATH/etcd/data/default.etcd
ETCD_WAL_DIR=$TOTAL_PATH/etcd/data/default.etcd
K8S_PATH=$TOTAL_PATH/k8s
POD_MANIFEST_PATH=$TOTAL_PATH/work
DOCKER_PATH=$TOTAL_PATH/docker
#DOCKER_BIN_PATH=$TOTAL_PATH/docker/bin #ubuntu 18 版本必须设置在/usr/bin 目录下面
DOCKER_BIN_PATH=/usr/bin
CNI_PATH=$TOTAL_PATH/cni
SOURCE_PATH=/usr/local/src # 远程服务器源码存放目录
KEEPALIVED_PATH=$TOTAL_PATH/keepalived
HAPROXY_PATH=$TOTAL_PATH/haproxy
# 设置工作端目录
HOST_PATH=`pwd`
# 设置工作端压缩包所在目录
TEMP_PATH=/tmp/source
#应用版本号
ETCD_VERSION=v3.3.18
K8S_VERSION=v1.17.0
LXCFS_VERSION=3.1.2
DOCKER_VERSION=19.03.5
CNI_VERSION=v0.8.3
IPTABLES_VERSION=1.6.2 #centos7,ubuntu18 版本需要升级  centos8, ubuntu19 不用升级
KEEPALIVED_VERSION=2.0.19
AUTOMAKE_VERSION=1.15.1 #KEEPALIVED 编译依赖使用
HAPROXY_VERSION=2.1.1
# 网络插件 选择 1、kube-router 2、kube-proxy+flannel 使用kube-router时external-ip 同网段不通需要做路由，kube-proxy 可以直接访问  
NET_PLUG=1
# 节点间互联网络接口名称flannel 指定网络接口
IFACE="eth0"
# K8S api 网络互联接口 多网卡请指定接口ansible_网卡接口名字.ipv4.address
API_IPV4=ansible_default_ipv4.address
# kubelet pod 网络互联接口 ansible_${IFACE}.ipv4.address 单网卡使用ansible_default_ipv4.address 多个网卡请指定使用的网卡名字
KUBELET_IPV4=ansible_default_ipv4.address 
# 证书相关配置
CERT_ST="GuangDong"
CERT_L="GuangZhou"
CERT_O="k8s"
CERT_OU="Qist"
CERT_PROFILE="kubernetes"
#数字证书时间及kube-controller-manager 签发证书时间
EXPIRY_TIME="87600h"
# K8S ETCD存储 目录名字
ETCD_PREFIX="/registry" 
# 配置etcd集群参数
#ETCD_SERVER_HOSTNAMES="\"k8s-master-01\",\"k8s-master-02\",\"k8s-master-03\""
#ETCD_SERVER_IPS="\"192.168.2.247\",\"192.168.2.248\",\"192.168.2.249\""
ETCD_MEMBER_1_IP="192.168.2.247"
ETCD_MEMBER_1_HOSTNAMES="k8s-master-01"
ETCD_MEMBER_2_IP="192.168.2.248"
ETCD_MEMBER_2_HOSTNAMES="k8s-master-02"
ETCD_MEMBER_3_IP="192.168.2.249"
ETCD_MEMBER_3_HOSTNAMES="k8s-master-03"
ETCD_SERVER_HOSTNAMES="\"${ETCD_MEMBER_1_HOSTNAMES}\",\"${ETCD_MEMBER_2_HOSTNAMES}\",\"${ETCD_MEMBER_3_HOSTNAMES}\""
ETCD_SERVER_IPS="\"${ETCD_MEMBER_1_IP}\",\"${ETCD_MEMBER_2_IP}\",\"${ETCD_MEMBER_3_IP}\""
# etcd 集群间通信的 IP 和端口
INITIAL_CLUSTER="${ETCD_MEMBER_1_HOSTNAMES}=https://${ETCD_MEMBER_1_IP}:2380,${ETCD_MEMBER_2_HOSTNAMES}=https://${ETCD_MEMBER_2_IP}:2380,${ETCD_MEMBER_3_HOSTNAMES}=https://${ETCD_MEMBER_3_IP}:2380"
# etcd 集群服务地址列表
ENDPOINTS=https://${ETCD_MEMBER_1_IP}:2379,https://${ETCD_MEMBER_2_IP}:2379,https://${ETCD_MEMBER_3_IP}:2379
#K8S events 存储ETCD 集群 1开启 默认关闭0
K8S_EVENTS=0
if [ ${K8S_EVENTS} == 1 ]; then
# etcd events集群配置
#ETCD_EVENTS_HOSTNAMES="\"k8s-node-01\",\"k8s-node-02\",\"k8s-node-03\""
#ETCD_EVENTS_IPS="\"192.168.2.250\",\"192.168.2.251\",\"192.168.2.252\""
ETCD_EVENTS_MEMBER_1_IP="192.168.2.250"
ETCD_EVENTS_MEMBER_1_HOSTNAMES="k8s-node-01"
ETCD_EVENTS_MEMBER_2_IP="192.168.2.251"
ETCD_EVENTS_MEMBER_2_HOSTNAMES="k8s-node-02"
ETCD_EVENTS_MEMBER_3_IP="192.168.2.252"
ETCD_EVENTS_MEMBER_3_HOSTNAMES="k8s-node-03"
ETCD_EVENTS_HOSTNAMES="\"${ETCD_EVENTS_MEMBER_1_HOSTNAMES}\",\"${ETCD_EVENTS_MEMBER_2_HOSTNAMES}\",\"${ETCD_EVENTS_MEMBER_3_HOSTNAMES}\""
ETCD_EVENTS_IPS="\"${ETCD_EVENTS_MEMBER_1_IP}\",\"${ETCD_EVENTS_MEMBER_2_IP}\",\"${ETCD_EVENTS_MEMBER_3_IP}\""
# etcd 集群间通信的 IP 和端口
INITIAL_EVENTS_CLUSTER="${ETCD_EVENTS_MEMBER_1_HOSTNAMES}=https://${ETCD_EVENTS_MEMBER_1_IP}:2380,${ETCD_EVENTS_MEMBER_2_HOSTNAMES}=https://${ETCD_EVENTS_MEMBER_2_IP}:2380,${ETCD_EVENTS_MEMBER_3_HOSTNAMES}=https://${ETCD_EVENTS_MEMBER_3_IP}:2380"
ENDPOINTS="${ENDPOINTS} --etcd-servers-overrides=/events#https://${ETCD_EVENTS_MEMBER_1_IP}:2379;https://${ETCD_EVENTS_MEMBER_2_IP}:2379;https://${ETCD_EVENTS_MEMBER_3_IP}:2379"
fi
#是否开启docker0 网卡 参数: doakcer0 none k8s集群建议不用开启，单独部署请设置值为docker0
NET_BRIDGE="none"
# 配置K8S集群参数
# 最好使用 当前未用的网段 来定义服务网段和 Pod 网段
# 服务网段，部署前路由不可达，部署后集群内路由可达(kube-proxy 保证)
SERVICE_CIDR="10.66.0.0/16"
# Pod 网段，建议 /12 段地址，部署前路由不可达，部署后集群内路由可达(网络插件 保证)
CLUSTER_CIDR="10.80.0.0/12"
# 服务端口范围 (NodePort Range)
NODE_PORT_RANGE="30000-65535"
# kubernetes 服务 IP (一般是 SERVICE_CIDR 中第一个IP)
CLUSTER_KUBERNETES_SVC_IP="10.66.0.1"
# 集群名字
CLUSTER_NAME=kubernetes
#集群域名
CLUSTER_DNS_DOMAIN="cluster.local"
#集群DNS
CLUSTER_DNS_SVC_IP="10.66.0.2"
# k8s vip ip
K8S_VIP_1_IP="192.168.3.12"
K8S_VIP_2_IP="192.168.3.13"
K8S_VIP_3_IP="192.168.3.14"
K8S_VIP_DOMAIN="api.k8s.niuke.tech"
#kube-apiserver port
SECURE_PORT=5443
# kube-apiserver vip port # 如果配置vip IP 请设置
K8S_VIP_PORT=6443
# kube-apiserver vip ip
KUBE_APISERVER="https://${K8S_VIP_DOMAIN}:${K8S_VIP_PORT}"
# RUNTIME_CONFIG v1.16 版本设置 低于v1.16 RUNTIME_CONFIG="api/all=true" 即可
RUNTIME_CONFIG="api/all=true"
#开启插件enable-admission-plugins #AlwaysPullImages 启用istio 不能自动注入需要手动执行注入
ENABLE_ADMISSION_PLUGINS="DefaultStorageClass,DefaultTolerationSeconds,LimitRanger,NamespaceExists,NamespaceLifecycle,NodeRestriction,OwnerReferencesPermissionEnforcement,PodNodeSelector,PersistentVolumeClaimResize,PodPreset,PodTolerationRestriction,ResourceQuota,ServiceAccount,StorageObjectInUseProtection,MutatingAdmissionWebhook,ValidatingAdmissionWebhook"
#禁用插件disable-admission-plugins 
DISABLE_ADMISSION_PLUGINS="DenyEscalatingExec,ExtendedResourceToleration,ImagePolicyWebhook,LimitPodHardAntiAffinityTopology,NamespaceAutoProvision,Priority,EventRateLimit,PodSecurityPolicy"
# 设置api 副本数
APISERVER_COUNT="3"
# 设置输出日志级别
LEVEL_LOG="2"
# api 突变请求最大数
MAX_MUTATING_REQUESTS_INFLIGHT="500"
# api 非突变请求的最大数目
MAX_REQUESTS_INFLIGHT="1500"
# 内存配置选项和node数量的关系，单位是MB： target-ram-mb=node_nums * 60
TARGET_RAM_MB="6000"
# kube-api-qps 默认50
KUBE_API_QPS="100"
#kube-api-burst 默认30
KUBE_API_BURST="100"
# pod-infra-container-image 地址
POD_INFRA_CONTAINER_IMAGE="docker.io/juestnow/pause-amd64:3.1"
# max-pods node 节点启动最多pod 数量
MAX_PODS=100
# 生成 EncryptionConfig 所需的加密 key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
#kube-apiserver 服务器IP列表 有更多的节点时请添加IP K8S_APISERVER_VIP="\"192.168.2.247\",\"192.168.2.248\",\"192.168.2.249\",\"192.168.2.250\",\"192.168.2.251\""
K8S_APISERVER_VIP="\"192.168.2.247\",\"192.168.2.248\",\"192.168.2.249\""
# 创建bootstrap配置
TOKEN_ID=$(head -c 6 /dev/urandom | md5sum | head -c 6)
TOKEN_SECRET=$(head -c 16 /dev/urandom | md5sum | head -c 16)
BOOTSTRAP_TOKEN=${TOKEN_ID}.${TOKEN_SECRET}
修改完成保存
```

# 创建新文件夹存放ansible 脚本及yml 脚本

```bash
# 到git clone 目录
mkdir k8s.v17.0
cd  k8s.v17.0
cp ../kubernetes.v1.17.sh ./
# 执行脚本
bash kubernetes.v1.17.sh
# 查看生成数据
[root@k8s-node-09 k8s.v17.0]# ls
README.md  cni.yml     environment.sh  haproxy.yml   keepalived.yml      kube-controller-manager.yml  kubeconfig   kubernetes.v1.17.sh  package.yml  yaml
cfssl      docker.yml  etcd.yml        iptables.yml  kube-apiserver.yml  kube-scheduler.yml           kubelet.yml  lxcfs.yml            roles
[root@k8s-node-09 k8s.v17.0]# tree
.
├── README.md
├── cfssl
│   ├── ca-config.json
│   ├── etcd
│   │   ├── etcd-ca-csr.json
│   │   ├── etcd-client.json
│   │   ├── etcd-server.json
│   │   ├── k8s-master-01.json
│   │   ├── k8s-master-02.json
│   │   └── k8s-master-03.json
│   ├── k8s
│   │   ├── aggregator.json
│   │   ├── k8s-apiserver-admin.json
│   │   ├── k8s-apiserver.json
│   │   ├── k8s-ca-csr.json
│   │   ├── k8s-controller-manager.json
│   │   ├── k8s-scheduler.json
│   │   └── kube-router.json
│   └── pki
│       ├── etcd
│       │   ├── etcd-ca-key.pem
│       │   ├── etcd-ca.csr
│       │   ├── etcd-ca.pem
│       │   ├── etcd-client-key.pem
│       │   ├── etcd-client.csr
│       │   ├── etcd-client.pem
│       │   ├── etcd-member-k8s-master-01-key.pem
│       │   ├── etcd-member-k8s-master-01.csr
│       │   ├── etcd-member-k8s-master-01.pem
│       │   ├── etcd-member-k8s-master-02-key.pem
│       │   ├── etcd-member-k8s-master-02.csr
│       │   ├── etcd-member-k8s-master-02.pem
│       │   ├── etcd-member-k8s-master-03-key.pem
│       │   ├── etcd-member-k8s-master-03.csr
│       │   ├── etcd-member-k8s-master-03.pem
│       │   ├── etcd-server-key.pem
│       │   ├── etcd-server.csr
│       │   └── etcd-server.pem
│       └── k8s
│           ├── aggregator-key.pem
│           ├── aggregator.csr
│           ├── aggregator.pem
│           ├── k8s-apiserver-admin-key.pem
│           ├── k8s-apiserver-admin.csr
│           ├── k8s-apiserver-admin.pem
│           ├── k8s-ca-key.pem
│           ├── k8s-ca.csr
│           ├── k8s-ca.pem
│           ├── k8s-controller-manager-key.pem
│           ├── k8s-controller-manager.csr
│           ├── k8s-controller-manager.pem
│           ├── k8s-scheduler-key.pem
│           ├── k8s-scheduler.csr
│           ├── k8s-scheduler.pem
│           ├── k8s-server-key.pem
│           ├── k8s-server.csr
│           ├── k8s-server.pem
│           ├── kube-router-key.pem
│           ├── kube-router.csr
│           └── kube-router.pem
├── cni.yml
├── docker.yml
├── environment.sh
├── etcd.yml
├── haproxy.yml
├── iptables.yml
├── keepalived.yml
├── kube-apiserver.yml
├── kube-controller-manager.yml
├── kube-scheduler.yml
├── kubeconfig
│   ├── admin.kubeconfig
│   ├── bootstrap.kubeconfig
│   ├── kube-controller-manager.kubeconfig
│   ├── kube-router.kubeconfig
│   └── kube-scheduler.kubeconfig
├── kubelet.yml
├── kubernetes.v1.17.sh
├── lxcfs.yml
├── package.yml
├── roles
│   ├── cni
│   │   ├── files
│   │   │   └── bin
│   │   │       ├── bandwidth
│   │   │       ├── bridge
│   │   │       ├── dhcp
│   │   │       ├── firewall
│   │   │       ├── flannel
│   │   │       ├── host-device
│   │   │       ├── host-local
│   │   │       ├── ipvlan
│   │   │       ├── loopback
│   │   │       ├── macvlan
│   │   │       ├── portmap
│   │   │       ├── ptp
│   │   │       ├── sbr
│   │   │       ├── static
│   │   │       ├── tuning
│   │   │       └── vlan
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   ├── docker
│   │   ├── files
│   │   │   └── bin
│   │   │       ├── containerd
│   │   │       ├── containerd-shim
│   │   │       ├── ctr
│   │   │       ├── docker
│   │   │       ├── docker-init
│   │   │       ├── docker-proxy
│   │   │       ├── dockerd
│   │   │       └── runc
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── containerd.service
│   │       ├── daemon.json
│   │       ├── docker.service
│   │       └── docker.socket
│   ├── etcd
│   │   ├── files
│   │   │   ├── bin
│   │   │   │   ├── etcd
│   │   │   │   └── etcdctl
│   │   │   └── ssl
│   │   │       ├── etcd-ca-key.pem
│   │   │       ├── etcd-ca.pem
│   │   │       ├── etcd-client-key.pem
│   │   │       ├── etcd-client.pem
│   │   │       ├── etcd-member-k8s-master-01-key.pem
│   │   │       ├── etcd-member-k8s-master-01.pem
│   │   │       ├── etcd-member-k8s-master-02-key.pem
│   │   │       ├── etcd-member-k8s-master-02.pem
│   │   │       ├── etcd-member-k8s-master-03-key.pem
│   │   │       ├── etcd-member-k8s-master-03.pem
│   │   │       ├── etcd-server-key.pem
│   │   │       └── etcd-server.pem
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── etcd
│   │       └── etcd.service
│   ├── haproxy
│   │   ├── files
│   │   │   └── haproxy-2.1.1.tar.gz
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── 49-haproxy.conf
│   │       ├── haproxy
│   │       ├── haproxy.conf
│   │       └── haproxy.service
│   ├── iptables
│   │   ├── files
│   │   │   └── iptables-1.6.2.tar.bz2
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   ├── keepalived
│   │   ├── files
│   │   │   ├── automake-1.15.1.tar.gz
│   │   │   └── keepalived-2.0.19.tar.gz
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── keepalived
│   │       ├── keepalived.conf
│   │       └── keepalived.service
│   ├── kube-apiserver
│   │   ├── files
│   │   │   ├── bin
│   │   │   │   └── kube-apiserver
│   │   │   ├── config
│   │   │   │   ├── audit-policy.yaml
│   │   │   │   └── encryption-config.yaml
│   │   │   └── ssl
│   │   │       ├── etcd
│   │   │       │   ├── etcd-ca.pem
│   │   │       │   ├── etcd-client-key.pem
│   │   │       │   └── etcd-client.pem
│   │   │       └── k8s
│   │   │           ├── aggregator-key.pem
│   │   │           ├── aggregator.pem
│   │   │           ├── k8s-ca.pem
│   │   │           ├── k8s-server-key.pem
│   │   │           └── k8s-server.pem
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── kube-apiserver
│   │       └── kube-apiserver.service
│   ├── kube-controller-manager
│   │   ├── files
│   │   │   ├── bin
│   │   │   │   └── kube-controller-manager
│   │   │   └── ssl
│   │   │       └── k8s
│   │   │           ├── k8s-ca-key.pem
│   │   │           ├── k8s-ca.pem
│   │   │           ├── k8s-controller-manager-key.pem
│   │   │           └── k8s-controller-manager.pem
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── kube-controller-manager
│   │       ├── kube-controller-manager.kubeconfig
│   │       └── kube-controller-manager.service
│   ├── kube-scheduler
│   │   ├── files
│   │   │   └── bin
│   │   │       └── kube-scheduler
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── kube-scheduler
│   │       ├── kube-scheduler.kubeconfig
│   │       └── kube-scheduler.service
│   ├── kubelet
│   │   ├── files
│   │   │   ├── bin
│   │   │   │   └── kubelet
│   │   │   └── ssl
│   │   │       └── k8s
│   │   │           └── k8s-ca.pem
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       ├── bootstrap.kubeconfig
│   │       ├── kubelet
│   │       └── kubelet.service
│   ├── lxcfs
│   │   ├── files
│   │   │   ├── lib
│   │   │   │   └── lxcfs
│   │   │   │       ├── liblxcfs.la
│   │   │   │       └── liblxcfs.so
│   │   │   ├── lxcfs
│   │   │   └── lxcfs.service
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   └── package
│       ├── files
│       ├── tasks
│       │   └── main.yml
│       └── templates
└── yaml
    ├── allow-lxcfs-tz-env.yaml
    ├── bootstrap-secret.yaml
    ├── kube-api-rbac.yaml
    ├── kube-router.yaml
    └── kubelet-bootstrap-rbac.yaml

75 directories, 179 files
```

# 查看生成README.md

```bash
## 特别说明： keepalived 部署 必须单个部署 三节点部署
ansible-playbook -i 192.168.2.247, keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA1 -e HA1_ID=100 -e HA2_ID=110 -e HA3_ID=120 -e STATE_3=MASTER
ansible-playbook -i 192.168.2.248, keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA2 -e HA1_ID=110 -e HA2_ID=120 -e HA3_ID=100 -e STATE_2=MASTER
ansible-playbook -i 192.168.2.249, keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA3 -e HA1_ID=120 -e HA2_ID=100 -e HA3_ID=110 -e STATE_1=MASTER
# 大于3节点部署 5节点
ansible-playbook -i "192.168.2.247", keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA1 -e HA1_ID=100 -e HA2_ID=110 -e HA3_ID=140 -e STATE_3=MASTER
ansible-playbook -i "192.168.2.248", keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA2 -e HA1_ID=110 -e HA2_ID=140 -e HA3_ID=130 -e STATE_2=MASTER
ansible-playbook -i "192.168.2.249", keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA3 -e HA1_ID=140 -e HA2_ID=100 -e HA3_ID=120 -e STATE_1=MASTER
ansible-playbook -i "192.168.2.250", keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA4 -e HA1_ID=130 -e HA2_ID=120 -e HA3_ID=110
ansible-playbook -i "192.168.2.251", keepalived.yml -e IFACE=eth0  -e ROUTER_ID=HA5 -e HA1_ID=120 -e HA2_ID=130 -e HA3_ID=100
# 安装说明文件执行就OK  node 节点安装ip 很多可以写如文件例如：
vi node
192.168.2.165
192.168.2.167
192.168.2.189
192.168.2.196
192.168.2.247
192.168.2.248
192.168.2.249
192.168.2.250
192.168.2.251
192.168.2.252
192.168.2.253
# 执行
ansible-playbook -i node xxx.yml 
[root@k8s-node-09 k8s.v17.0]# cat README.md
########## mkdir -p /root/.kube
##########复制admin kubeconfig 到root用户作为kubectl 工具默认密钥文件
########## \cp -pdr /opt/k8s/k8s.v17.0/kubeconfig/admin.kubeconfig /root/.kube/config
###################################################################################
##########  ansible 及ansible-playbook 单个ip ip结尾一点要添加“,”符号 ansible-playbook -i 192.168.0.1, xxx.yml
##########  source /opt/k8s/k8s.v17.0/environment.sh 设置环境变量生效方便后期新增证书等
##########  etcd 部署 ansible-playbook -i "192.168.2.247","192.168.2.248","192.168.2.249" etcd.yml
##########  etcd EVENTS 部署 ansible-playbook -i , events-etcd.yml
##########  kube-apiserver 部署 ansible-playbook -i "192.168.2.247","192.168.2.248","192.168.2.249", kube-apiserver.yml
##########  haproxy 部署 ansible-playbook -i "192.168.2.247","192.168.2.248","192.168.2.249", haproxy.yml
##########  keepalived 节点IP "192.168.2.247","192.168.2.248","192.168.2.249" 安装keepalived使用IP 如果大于三个节点安装keepalived 记得HA1_ID 唯一的也就是priority的值
##########  keepalived 也可以全部部署为BACKUP STATE_x 可以使用默认值 IFACE 网卡名字默认eth0 ROUTER_ID 全局唯一ID   HA1_ID为priority值
##########  keepalived 部署 节点1 ansible-playbook -i 节点ip1, keepalived.yml -e IFACE=eth0 -e ROUTER_ID=HA1 -e HA1_ID=100 -e HA2_ID=110 -e HA3_ID=120 -e STATE_3=MASTER
##########  keepalived 部署 节点2 ansible-playbook -i 节点ip2, keepalived.yml -e IFACE=eth0 -e ROUTER_ID=HA2 -e HA1_ID=110 -e HA2_ID=120 -e HA3_ID=100 -e STATE_2=MASTER
##########  keepalived 部署 节点3 ansible-playbook -i 节点ip3, keepalived.yml -e IFACE=eth0 -e ROUTER_ID=HA3 -e HA1_ID=120 -e HA2_ID=100 -e HA3_ID=110 -e STATE_1=MASTER
##########  kube-controller-manager kube-scheduler  ansible-playbook -i "192.168.2.247","192.168.2.248","192.168.2.249", kube-controller-manager.yml kube-scheduler.yml
##########  部署完成验证集群 kubectl cluster-info  kubectl api-versions  kubectl get cs 1.16 kubectl 显示不正常
##########  提交bootstrap 跟授权到K8S 集群 kubectl apply -f /opt/k8s/k8s.v17.0/yaml/bootstrap-secret.yaml
##########  提交授权到K8S集群 kubectl apply -f /opt/k8s/k8s.v17.0/yaml/kubelet-bootstrap-rbac.yaml kubectl apply -f /opt/k8s/k8s.v17.0/yaml/kube-api-rbac.yaml
##########  系统版本为centos7 或者 ubuntu18 请先升级 iptables ansible-playbook -i  要安装node ip列表, iptables.yml
##########  安装K8S node 使用kube-router ansible部署 ansible-playbook -i 要安装node ip列表 package.yml lxcfs.yml docker.yml kubelet.yml
##########  安装K8S node 使用 flannel 网络插件ansible部署ansible-playbook -i 要安装node ip列表 package.yml lxcfs.yml docker.yml kubelet.yml kube-proxy.yml
##########  部署自动挂载日期与lxcfs 到pod的 PodPreset  kubectl apply -f /opt/k8s/k8s.v17.0/yaml/allow-lxcfs-tz-env.yaml -n kube-system  " kube-system 命名空间名字"PodPreset 只是当前空间生效所以需要每个命名空间执行
##########  查看node 节点是否注册到K8S kubectl get node kubectl get csr 如果有节点 kube-router 方式部署 kubectl apply -f /opt/k8s/k8s.v17.0/yaml/kube-router.yaml 等待容器部署完成查看node ip a | grep kube-bridge
##########  flannel 网络插件部署 kubectl apply -f /opt/k8s/k8s.v17.0/yaml/flannel.yaml 等待容器部署完成查看node 节点网络 ip a| grep flannel.1
##########  给 master ingress 添加污点 防止其它服务使用这些节点:kubectl taint nodes  k8s-master-01 node-role.kubernetes.io/master=:NoSchedule kubectl taint nodes  k8s-ingress-01 node-role.kubernetes.io/ingress=:NoSchedule
##########  calico 网络插件部署 50节点内 wget https://docs.projectcalico.org/v3.10/manifests/calico.yaml  大于50节点 wget https://docs.projectcalico.org/v3.10/manifests/calico-typha.yaml
########## 如果cni配置没放到默认路径请创建软链 ln -s /apps/cni/etc /etc/cni 同时修改yaml hostPath路径 同时修改CALICO_IPV4POOL_CIDR 参数为 10.80.0.0/12 CALICO_IPV4POOL_IPIP: Never 启用bgp模式
##########  windows 证书访问 openssl pkcs12 -export -inkey k8s-apiserver-admin-key.pem -in k8s_apiserver-admin.pem -out client.p12
########## kubectl proxy --port=8001 &  把kube-apiserver 端口映射成本地 8001 端口
########## 查看kubelet节点配置信息 NODE_NAME="k8s-node-04"; curl -sSL "http://localhost:8001/api/v1/nodes/${NODE_NAME}/proxy/configz" | jq '.kubeletconfig|.kind="KubeletConfiguration"|.apiVersion="kubelet.config.k8s.io/v1beta1"' > kubele
t_configz_${NODE_NAME}
```

# 创建新证书例如：kubernetes-dashboard或者kubeconfig

```
# 生效环境变量
source /opt/k8s/k8s.v17.0/environment.sh 
cat << EOF | tee ${HOST_PATH}/cfssl/k8s/kubernetes-dashboard.json
{
  "CN": "kubernetes-dashboard",
  "hosts": [""], 
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
            "C": "CN",
            "ST": "$CERT_ST",
            "L": "$CERT_L",
            "O": "$CERT_O",
            "OU": "$CERT_OU"
    }
  ]
}
EOF
# 生成新的证书
cfssl gencert \
        -ca=${HOST_PATH}/cfssl/pki/k8s/k8s-ca.pem \
        -ca-key=${HOST_PATH}/cfssl/pki/k8s/k8s-ca-key.pem \
        -config=${HOST_PATH}/cfssl/ca-config.json \
        -profile=${CERT_PROFILE} \
         ${HOST_PATH}/cfssl/k8s/kubernetes-dashboard.json | \
         cfssljson -bare ./kubernetes-dashboard
```

# 删除集群

```bash
 kubectl delete deployments  --all -A
 kubectl delete daemonsets  --all -A
 kubectl delete statefulsets --all  -A
 ansible -i node all -m shell -a "systemctl stop kubelet"
 ansible -i node all -m shell -a "systemctl stop docker"
 ansible -i node all -m shell -a "systemctl stop kube-proxy"
 ansible -i node all -m shell -a "systemctl stop containerd"
 ansible -i node all -m shell -a "systemctl stop kube-scheduler"
 ansible -i node all -m shell -a "systemctl stop kube-controller-manager"
 ansible -i node all -m shell -a "systemctl stop kube-apiserver"
 ansible -i node all -m shell -a "systemctl stop etcd"
 ansible -i node all -m shell -a "systemctl stop haproxy"
 ansible -i node all -m shell -a "systemctl stop keepalived"
 ansible -i node all -m shell -a "umount /apps/docker/root/netns/default"
 # 目录更改成自己环境路径 如果docker 跟其它应用部署一起就直接可以删除
ansible -i node all -m shell -a "rm -rf /apps/*"
ansible -i node all -m shell -a "rm -rf /etc/cni"
ansible -i node all -m shell -a "rm -rf /etc/docker"
ansible -i node all -m shell -a "rm -rf /etc/containerd"
ansible -i node all -m shell -a "rm -f  /usr/bin/docker*"
ansible -i node all -m shell -a "rm -f  /usr/bin/containerd*"
ansible -i node all -m shell -a "rm -f  /usr/bin/ctr"
ansible -i node all -m shell -a "rm -f  /usr/bin/runc"
# 关闭开机启动
 ansible -i node all -m shell -a "systemctl disable kubelet"
 ansible -i node all -m shell -a "systemctl disable docker"
 ansible -i node all -m shell -a "systemctl disable kube-proxy"
 ansible -i node all -m shell -a "systemctl disable containerd"
 ansible -i node all -m shell -a "systemctl disable kube-scheduler"
 ansible -i node all -m shell -a "systemctl disable kube-controller-manager"
 ansible -i node all -m shell -a "systemctl disable kube-apiserver"
 ansible -i node all -m shell -a "systemctl disable etcd"
 ansible -i node all -m shell -a "systemctl disable haproxy"
 ansible -i node all -m shell -a "systemctl disable keepalived"
# 删除开机启动文件
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/kubelet.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/docker.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/docker.socket"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/kube-proxy.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/containerd.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/kube-scheduler.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/kube-controller-manager.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/kube-apiserver.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/etcd.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/haproxy.service"
 ansible -i node all -m shell -a "rm -f /lib/systemd/system/keepalived.service"
```