def label = "slave-${UUID.randomUUID().toString()}"
podTemplate(label: label, serviceAccount: 'jenkins2', namespace: 'kube-ops', containers: [
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'registry.cn-hangzhou.aliyuncs.com/k8s_xzb/kubectl:v1.15.0', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'maven', image: 'maven:3.6-alpine', command: 'cat', ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
    node(label) {
        // 这里面放你以前的代码，其他地方需要用指定容器的，看下面的示例

        stage('Build') {
          echo "2.maven打包镜像"
          // 只有maven容器有mvn，所以必须指明
          container('maven') {
            //这里放你原来的mvn命令
          }
        }
        stage('Build And Push') {
            // 只有docker容器有docker，所有docker命令运行在docker容器
            container('docker') {
               // 这里放你原来的docker相关的代码
            }
        }
        stage('Deploy') {
            // 只有kubectl容器有kubectl命令，所有kubectl命令运行在kubectl容器
            container('kubectl') {
                // 这里放你原来的kubectl相关代码
            }
        }
    }
}