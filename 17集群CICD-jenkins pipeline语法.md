[toc]

# jenkins pipeline用法

## 1. 声明式与脚本式语法

写在jenkins任务里的就是脚本式，写在Jenkinsfile里的就是声明式(这是错的)。

- 声明式比脚本式支持更多的语法功能，可读性更强

- 声明式必须在一个`pipeline`代码块里，如下简单例子

- 脚本式和声明式写的Jenkinfile都可以放在源码那里一起做版本管理

  脚本化流水线, 与[[declarative-pipeline\]](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-pipeline)一样的是, 是建立在底层流水线的子系统上的。与声明式不同的是, 脚本化流水线实际上是由 [Groovy](http://groovy-lang.org/syntax.html)构建的通用 DSL [[2](https://jenkins.io/zh/doc/book/pipeline/syntax/#_footnotedef_2)]。 Groovy 语言提供的大部分功能都可以用于脚本化流水线的用户。这意味着它是一个非常有表现力和灵活的工具，可以通过它编写持续交付流水线。

  ### 流控制

  脚本化流水线从 `Jenkinsfile` 的顶部开始向下串行执行, 就像 Groovy 或其他语言中的大多数传统脚本一样。 因此，提供流控制取决于 Groovy 表达式, 比如 `if/else` 条件, 例如:

  ```groovy
  Jenkinsfile (Scripted Pipeline)
  node {
      stage('Example') {
          if (env.BRANCH_NAME == 'master') {
              echo 'I only execute on the master branch'
          } else {
              echo 'I execute elsewhere'
          }
      }
  }
  ```

  另一种方法是使用Groovy的异常处理支持来管理脚本化流水线流控制。当 [步骤](https://jenkins.io/zh/doc/book/pipeline/syntax/#scripted-steps) 失败 ，无论什么原因，它们都会抛出一个异常。处理错误的行为必须使用Groovy中的 `try/catch/finally` 块 , 例如:

  ```groovy
  Jenkinsfile (Scripted Pipeline)
  node {
      stage('Example') {
          try {
              sh 'exit 1'
          }
          catch (exc) {
              echo 'Something failed, I should sound the klaxons!'
              throw
          }
      }
  }
  ```

  

## 2. 简单例子

```groovy
pipeline {
  //Execute this Pipeline or any of its stages, on any available agent
  agent any
  stage('拉代码') {
    echo "拉取代码"
  }
}
```

## 3. agent

用于指明pipeline在哪个jenkins环境运行。必须定义在`pipeline`的顶部(必需)，但`stage`里也可以使用它(非必需)

Parameters:

- any

  在任意可用的agent上执行pipeline，用法`agent any`

- none

  当顶部agent定义为none时，每个stage必须定义其agent，用法`agent none`

- label

  通过label来选择agent

  ```groovy
  pipeline {
    agent {
      label 'labelName'
    }
  }
  ```

- node

  通过node来选择agent，可使用更多选项，如`customWorkspace`

  ```groovy
  agent { 
    node { 
      label 'labelName' 
    } 
  }
  //等同于下面的效果
  agent {
    label 'labelName'
  }
  ```

- docker

  定义agent使用的镜像

  ```groovy
  agent {
      docker {
          image 'maven:3-alpine'
          label 'my-defined-label'
          args  '-v /tmp:/tmp'
      }
  }
  ```

  `docker`同样接受`registryUrl`和`registryCredentialsId`参数，如下

  ```groovy
  agent {
      docker {
          image 'myregistry.com/node'
          label 'my-defined-label'
          registryUrl 'https://myregistry.com/'
          registryCredentialsId 'myPredefinedCredentialsInJenkins'
      }
  }
  ```

  

- dockerfile

  如果使用根目录的Dockerfile，如下即可

  ```groovy
  agent {
    dockerfile true
  }
  ```

  如果Dockerfile在别的地方，如下

  ```groovy
  agent {
      // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
      dockerfile {
          filename 'Dockerfile.build'
          dir 'build'
          label 'my-defined-label'
          additionalBuildArgs  '--build-arg version=1.0.2'
          args '-v /tmp:/tmp'
      }
  }
  ```

  

- kubernetes

  如下的`Jenkinsfile`必须从`Multibranch Pipeline`或`Pipeline from SCM`加载

  ```groovy
  agent {
      kubernetes {
          label podlabel
          yaml """
  kind: Pod
  metadata:
    name: jenkins-slave
  spec:
    containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      imagePullPolicy: Always
      command:
      - /busybox/cat
      tty: true
      volumeMounts:
        - name: aws-secret
          mountPath: /root/.aws/
        - name: docker-registry-config
          mountPath: /kaniko/.docker
    restartPolicy: Never
    volumes:
      - name: aws-secret
        secret:
          secretName: aws-secret
      - name: docker-registry-config
        configMap:
          name: docker-registry-config
  """
     }
  }
  ```

  

## 4. 通用的选项

- label

  string类型

  作用域：This option is valid for `node`, `docker` and `dockerfile`, and is required for `node`.

- customWorkspace

  string类型

  ```groovy
  agent {
      node {
          label 'my-defined-label'
          customWorkspace '/some/other/path'
      }
  }
  ```

  作用域：This option is valid for `node`, `docker` and `dockerfile`.

- reuseNode

  boolean类型，默认是false

  作用域：This option is valid for `docker` and `dockerfile`, and only has an effect when used on an `agent` for an individual `stage`.

- args

  string类型

  作用域：This option is valid for `docker` and `dockerfile`.

## 3. `secret text`、`username and password`和`secret file`的调用

```groovy
pipeline {
  agent {
    // Define agent details here
  }
  environment {
    // 声明，这里声明，作用域是整个pipeline，如果stage里声明，仅作用于stage
		DOCKER_HUB_CREDS = credentials('dockerHub')
  }
  // 调用
  stage('docker login') {
    echo '登陆dockerhub'
    // 对比使用单引号和双引号调用的区别，注意控制台输出
    sh '''
    		echo ${DOCKER_HUB_CREDS_USR}
    		echo ${DOCKER_HUB_CREDS_PSW}
    '''
    sh """
    		echo ${DOCKER_HUB_CREDS_USR}
    		echo ${DOCKER_HUB_CREDS_PSW}
    """
    sh "docker login ${DOCKER_HUB_CREDS_USR} ${DOCKER_HUB_CREDS_PSW}"
    // sh由Pipeline:Nodes and Processes plugin提供
  }
  stage('docker push') {
    echo "docker push"
    sh "docker push dukanghub/${IMAGE_NAME}:${IMAGE_TAG}"
  }

}
```

凭据绑定功能由插件[Credentials Binding plugin](https://plugins.jenkins.io/credentials-binding)提供

## 4. SSH-keys与证书的调用

**SSH User Private Key example**

```groovy
withCredentials(bindings: [sshUserPrivateKey(credentialsId: 'jenkins-ssh-key-for-abc', \
                                             keyFileVariable: 'SSH_KEY_FOR_ABC', \
                                             passphraseVariable: '', \
                                             usernameVariable: '')]) {
  // some block
}
```

 `passphraseVariable` 和 `usernameVariable` 选项可以被删除，因为这里用不上，如果不为空，则不能删除.

**Certificate example**

```groovy
withCredentials(bindings: [certificate(aliasVariable: '', \
                                       credentialsId: 'jenkins-certificate-for-xyz', \
                                       keystoreVariable: 'CERTIFICATE_FOR_XYZ', \
                                       passwordVariable: 'XYZ-CERTIFICATE-PASSWORD')]) {
  // some block
}
```

`aliasVariable` and `passwordVariable` 可以被删除.

## 5. 环境变量是可以自定义的

```groovy
enkinsfile (Declarative Pipeline)
pipeline {
    agent {
        label '!windows'
    }
		// 自定义环境变量
    environment {
        DISABLE_AUTH = 'true'
        DB_ENGINE    = 'sqlite'
    }

    stages {
        stage('Build') {
            steps {
              	// 环境变量的使用
                echo "Database engine is ${DB_ENGINE}"
                echo "DISABLE_AUTH is ${DISABLE_AUTH}"
                sh 'printenv'
            }
        }
    }
}
```

**注意：**pipeline定义的任何阶段的变量，可通过使用全局变量env.变量名获取

动态设置环境变量

```groovy
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any 
    environment {
        // Using returnStdout
        CC = """${sh(
                returnStdout: true,
                script: 'echo "clang"'
            )}""" 
        // Using returnStatus
        EXIT_STATUS = """${sh(
                returnStatus: true,
                script: 'exit 1'
            )}"""
    }
    stages {
        stage('Example') {
            environment {
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

**注：**

- agent必须设置在pipeline的顶部，且不能为空。否则将会失败
- agent和node是两个不同的模块
- 使用`returnStdout`，结尾会自动追加一个空格，可使用`.trim()`去除此空格

## 6. 使用`post`记录pipeline结果

```groovy
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Test') {
            steps {
                sh './gradlew check'
            }
        }
    }
		// 跑完上面的步骤后，记录结果文件和报告
    post {
      	// always表示总是记录，
        always {
            archiveArtifacts artifacts: 'build/libs/**/*.jar', fingerprint: true
            junit 'build/reports/**/*.xml'
        }
    }
}
```

##### 条件

- `always`

  运行，无论Pipeline运行的完成状态如何。

- `changed`

  只有当前Pipeline运行的状态与先前完成的Pipeline的状态不同时，才能运行。

- `failure`

  仅当当前Pipeline处于“失败”状态时才运行，通常在Web UI中用红色指示表示。

- `success`

  仅当当前Pipeline具有“成功”状态时才运行，通常在具有蓝色或绿色指示的Web UI中表示。

- `unstable`

  只有当前Pipeline具有“不稳定”状态，通常由测试失败，代码违例等引起，才能运行。通常在具有黄色指示的Web UI中表示。

- `aborted`

  只有当前Pipeline处于“中止”状态时，才会运行，通常是由于Pipeline被手动中止。通常在具有灰色指示的Web UI中表示。

## 7. 构建后的清理工作和通知

```groovy
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('No-op') {
            steps {
                sh 'ls'
            }
        }
    }
    post {
        always {
            echo 'One way or another, I have finished'
            deleteDir() /* clean up our workspace */
        }
        success {
            echo 'I succeeeded!'
        }
        unstable {
            echo 'I am unstable :/'
        }
        failure {
            echo 'I failed :('
          	mail to: 'team@example.com',
              	subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
              	body: "Something is wrong with ${env.BUILD_URL}"
        }
        changed {
            echo 'Things were different before...'
        }
    }
}
```

## 8. 使用`input`人工干预pipeline

有些时候需要人工确认，所以需要通过`input`来接收人工输入的信息来选择下一步如何走

```groovy
stage('Deploy') {
  echo "5. Deploy Stage"
  def userInput = input(
    id: 'userInput',
    message: 'Choose a deploy environment',
    parameters: [
      [
        $class: 'ChoiceParameterDefinition',
        choices: "Dev\nQA\nProd",
        name: 'Env'
      ]
    ]
  )
  echo "This is a deploy step to ${userInput}"
  sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
  sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s.yaml"
  if (userInput == "Dev") {
    // deploy dev stuff
  } else if (userInput == "QA"){
    // deploy qa stuff
  } else {
    // deploy prod stuff
  }
  sh "kubectl apply -f k8s.yaml"
}
```

## 9. 自动构建java-app

```groovy
pipeline {
    agent {
      	// 启动一个maven:3-alpine镜像的容器作为jenkins的agent，用来编译java
        docker {
            image 'maven:3-alpine' 
          	// 映射这个目录是为了每次编译不用重复下载依赖包
            args '-v /root/.m2:/root/.m2' 
        }
    }
    stages {
        stage('Build') { 
            steps {
                sh 'mvn -B -DskipTests clean package' 
            }
        }
      	stage('Test') { 
            steps {
                sh 'mvn test' 
              	//测试完成后，会在容器项目目录target/surefire-reports下生成一个xml的报告
            }
            post {
                always {
                  	//junit由JUit Plugin提供，将上面生成的报告通过jenkins暴露出来访问
                    junit 'target/surefire-reports/*.xml' 
                }
            }
        }
      	stage('Deliver') { 
            steps {
              	//这个脚本的相对路径是你源码路径，使用脚本实现复杂功能可使pipeline更简洁
                sh './jenkins/scripts/deliver.sh' 
            }
        }
    }
}
```

## 10. 使用多agent

```groovy
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps {
                checkout scm
                sh 'make'
                stash includes: '**/target/*.jar', name: 'app' 
            }
        }
        stage('Test on Linux') {
            agent { 
                label 'linux'
            }
            steps {
                unstash 'app' 
                sh 'make check'
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
        stage('Test on Windows') {
            agent {
                label 'windows'
            }
            steps {
                unstash 'app'
                bat 'make check' 
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
    }
}
```

## 11. 可选step参数

```groovy
// 以下两个效果相同
git url: 'git://example.com/amazing-project.git', branch: 'master'
git([url: 'git://example.com/amazing-project.git', branch: 'master'])

// 以下两个效果相同
sh 'echo hello' /* short form  */
sh([script: 'echo hello'])  /* long form */
```

## 12. 多任务平行执行

```groovy
Jenkinsfile (Scripted Pipeline)
stage('Build') {
    /* .. snip .. */
}

stage('Test') {
    parallel linux: {
        node('linux') {
            checkout scm
            try {
                unstash 'app'
                sh 'make check'
            }
            finally {
                junit '**/target/*.xml'
            }
        }
    },
    windows: {
        node('windows') {
            /* .. snip .. */
        }
    }
}
```

上面例子中，`linux`和`windows`的两个Test任务将同时进行，避免等待。

## 13.  `stash`保留

```groovy
options {
  	// 不带数量，默认是1
    preserveStashes() 
    // 或者指定数量1-50
    preserveStashes(buildCount: 5) 
}
```

## 14. 自定义镜像

```groovy
node {
    checkout scm
		//build方法默认是调用源码根目录的Dockerfile构建镜像，也可以手动指定dockerfile位置
    def customImage = docker.build("my-image:${env.BUILD_ID}")

    customImage.inside {
        sh 'make test'
    }
  	//自定义镜像可以提交到dockerhub或自己私仓
  	customImage.push()
}
```

