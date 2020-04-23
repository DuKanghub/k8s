node('test-jnlp') {

    stage('回滚') {
        echo "1.选择回滚版本"
        def BRANCH_TO_BUILD = input(
        id: "tttt",
        message: 'Choose a roll version',
        parameters: [
            [
                $class: 'ChoiceParameterDefinition',
                choices: "回滚至上一版本\n回滚至指定版本",
                name: 'BRANCH_TO_BUILD'
            ]
        ]
        )
        echo "${BRANCH_TO_BUILD}"
        script {
            if ("${BRANCH_TO_BUILD}" == "回滚至指定版本") {
                withCredentials([usernamePassword(credentialsId: 'hub_xzb', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                    tag_list = sh(returnStdout: true, script: 'curl -s -u "${dockerHubUser}:${dockerHubPassword}" -X GET "https://harbor.xiazaibei.com/api/repositories/test/nginx-alpine/tags"|jq ".[].name"|cut -f2 -d\\"|sort -r|head -10')
                }
                echo "${env.tag_list}"
                def BRANCH_TO_BUILD1 = input(
                message: 'Choose a deploy environment',

                parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "${tag_list}",
                    name: 'BRANCH_TO_BUILD1'
                ]
                ]
                )
                checkout scm
                sh "sed -i 's/<BUILD_TAG>/${BRANCH_TO_BUILD1}/' k8s.yaml"
                sh "kubectl apply -f k8s.yaml --record"
                sh "kubectl apply -f nginx_ingress.yaml"

            
            

            } else {
                sh "kubectl rollout -n default undo deployments/nginx-alpine-html"
            }
        }

        
        
    }
}