[toc]

# jenkins+gitlab+微服务发布+k8s发布

## 1. 配置jenkins pipeline

 ![jenkins+gitlab+å¾®æå¡åå¸+k8såå¸](https://s1.51cto.com/images/blog/201905/24/e1edb23566797e1a0208f88c94205c37.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=) 

## 2. 脚本式pipeline语法

```groovy
node {
  try {
  
    stage('代码拉取') {
    	git credentialsId: 'xiongxj', url: 'git@git.bqjr.club:xinjiang.xiong/oam.git'
    }
    
    stage('项目构建') {
       sh " /opt/software/apache-maven-3.6.0/bin/mvn clean package" 
    }
    def regPrefix = 'k8s.harbor.maimaiti.site/oam/' 
    stage('镜像构建') {
      docker.withRegistry('http://k8s.harbor.maimaiti.site/','k8sharbor'){
           if ("${MODULE}".contains('oamboot-eurekaServer')){
             dir('oamboot-eureka-server') {
                  def imageName = docker.build("${regPrefix}oam-eurekaserver:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-eurekaserver:V1.0-${env.BUILD_ID}"                
              }
              }
           if ("${MODULE}".contains('oambootActiviti')){
             dir('oamboot-activiti') {
                  def imageName = docker.build("${regPrefix}oam-activiti:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-activiti:V1.0-${env.BUILD_ID}"                
              }
              }       
           if ("${MODULE}".contains('oamboot-applyBasicOperation')){
             dir('oamboot-applyBasicOperation') {
                  def imageName = docker.build("${regPrefix}apply-basicoperation:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}apply-basicoperation:V1.0-${env.BUILD_ID}"                
              }
              }                   
           if ("${MODULE}".contains('oamboot-applyDB')){
             dir('oamboot-applyDB') {
                  def imageName = docker.build("${regPrefix}oam-applydb:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-applydb:V1.0-${env.BUILD_ID}"                
              }
              }                   
           if ("${MODULE}".contains('oamboot-applyOperation')){
             dir('oamboot-applyOperation') {
                  def imageName = docker.build("${regPrefix}apply-operation:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}apply-operation:V1.0-${env.BUILD_ID}"                
              }
              }                   
           if ("${MODULE}".contains('oamboot-basic')){
             dir('oamboot-basic') {
                  def imageName = docker.build("${regPrefix}oam-basic:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-basic:V1.0-${env.BUILD_ID}"                
              }
              }                   
           if ("${MODULE}".contains('oamboot-cmdb')){
             dir('oamboot-cmdb') {
                  def imageName = docker.build("${regPrefix}oamboot-cmdb:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oamboot-cmdb:V1.0-${env.BUILD_ID}"                
              }
              }   
           if ("${MODULE}".contains('oamboot-email')){
             dir('oamboot-email') {
                  def imageName = docker.build("${regPrefix}oam-email:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-email:V1.0-${env.BUILD_ID}"                
              }
              }   
           if ("${MODULE}".contains('oamboot-file')){
             dir('oamboot-file') {
                  def imageName = docker.build("${regPrefix}oam-file:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-file:V1.0-${env.BUILD_ID}"                
              }
              }   
          if ("${MODULE}".contains('oamboot-zuul')){
             dir('oamboot-zuul') {
                  def imageName = docker.build("${regPrefix}oam-api-getaway-zuul:V1.0-${env.BUILD_ID}")
                  imageName.push("V1.0-${env.BUILD_ID}")
                  //imageName.push("latest")
                  sh "/usr/bin/docker rmi ${regPrefix}oam-api-getaway-zuul:V1.0-${env.BUILD_ID}"                
              }
              }
             }                
        }
    stage('重启应用'){
         if ("${MODULE}".contains('oamboot-eurekaServer')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oam-EurekaServer.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oam-EurekaServer.yml  --record "    
            }
         if ("${MODULE}".contains('oambootActiviti')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oam-activiti.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oam-activiti.yml  --record "                  
            }       
         if ("${MODULE}".contains('oamboot-applyBasicOperation')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/apply-basicoperation.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/apply-basicoperation.yml --record " 
            }                   
         if ("${MODULE}".contains('oamboot-applyDB')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oam-applydb.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oam-applydb.yml  --record " 
            }                   
         if ("${MODULE}".contains('oamboot-applyOperation')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/apply-operation.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/apply-operation.yml  --record " 
            }                   
         if ("${MODULE}".contains('oamboot-basic')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oam-basic.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oam-basic.yml  --record "   
            }                   
         if ("${MODULE}".contains('oamboot-cmdb')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oamboot-cmdb.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oamboot-cmdb.yml  --record "    
            }   
         if ("${MODULE}".contains('oamboot-email')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oamboot-email.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oamboot-email.yml  --record "   
            }   
         if ("${MODULE}".contains('oamboot-file')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oam-file.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oam-file.yml  --record "    
            }   
        if ("${MODULE}".contains('oamboot-zuul')){
            sh "sed -i \'s/latest/V1.0-${env.BUILD_ID}/g\' yaml/oamboot-zuul.yml"
            sh "/usr/local/bin/kubectl   apply  -f         yaml/oamboot-zuul.yml  --record "                
        }            

    }       
  }catch (any) {
  currentBuild.result = 'FAILURE'
  throw any}

}
```

