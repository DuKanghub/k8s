[toc]

**GlusterFS 存储的 StorageClass 配置**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast                            #---SorageClass 名称
provisioner: kubernetes.io/glusterfs    #---标识 provisioner 为 GlusterFS
parameters:
  resturl: "http://10.10.249.63:8080"   
  restuser: "admin"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "none"  #---分布巻模式，不提供备份，正式环境切勿用此模式
```

