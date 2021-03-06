pipeline {
    agent any

    parameters {
       // string(defaultValue: 'v2', description: '', name: 'buildVersion')
        choice(
            choices: 'Rollingupdate\nBlue-Green',
            description: 'Deployment Type',
            name: 'REQUESTED_ACTION')
    }
    environment {
    // Environment variable identifiers need to be both valid bash variable
    // identifiers and valid Groovy variable identifiers. If you use an invalid
    // identifier, you'll get an error at validation time.
    // Right now, you can't do more complicated Groovy expressions or nesting of
    // other env vars in environment variable values, but that will be possible
    // when https://issues.jenkins-ci.org/browse/JENKINS-41748 is merged and
    // released.
        service = "nodejs-service"
        deployment = "nodejs"
        DEPLOYMENTFILE = "deploy-green.yml"
        VERSION= "${BUILD_ID}"
        image= "durgaoct1989/nodejs-app-demo"
     }
    
    stages {
        stage("build") {
            steps {
                //sh 'rm -rf Kubernetes-Demo'
                //sh 'git clone https://github.com/HM-demo/Kubernetes-Demo'
                deleteDir()
                checkout scm
            }
        }
        stage("Docker image build") {
            steps {
                sh 'sudo docker build -t nodejs-image-new .'
            }
        }
        stage("Docker image tag") {
            steps {
                sh 'echo ${image}'
                sh 'sudo  docker tag nodejs-image-new ${image}:${VERSION}'
                //sh 'sudo  docker tag nodejs-image-new ${image}'
            }
        }
        stage("Docker image push") {
            steps {
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId:'docker-demo-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
                sh '''sudo docker login -u $USERNAME -p $PASSWORD
                sudo docker push ${image}:${VERSION}
                #cd Kubernetes-Demo
                sed -i -e 's/nodejs-app-demo/nodejs-app-demo:'${VERSION}'/g' patch.yaml
                '''
                }     
            }
        }
        
        stage("Rollingupdate Deployment") {
             when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Rollingupdate' }
            }
            steps {
                sh '''
                # cd Kubernetes-Demo
                kubectl patch deployment ${deployment} --patch "$(cat patch.yaml)"
                '''
            }
        }
        stage("Blue-green Deployment") {
            when {
                // Only say hello if a "greeting" is requested
                expression { params.REQUESTED_ACTION == 'Blue-Green' }
            }
            steps {
                sh '''
                #cd Kubernetes-Demo
                curr_env=`kubectl get svc service-blue-green -o jsonpath="{.spec.selector.color}"`
                if [ $curr_env = "blue" ];then new_env="green";else new_env="blue";fi
                 sed -i -e 's/nodejs-app-demo/nodejs-app-demo:'${VERSION}'/g' deploy-${new_env}.yaml
                sed -i -e 's/envColor/"'${new_env}'"/g' patch-svc.yaml
                kubectl create -f deploy-${new_env}.yaml
                kubectl patch svc service-blue-green -p "$(cat patch-svc.yaml)"
                sleep 60
                SERVICE_IP=`kubectl get svc service-blue-green -o jsonpath="{.status.loadBalancer.ingress[0].*}"`
                response=$(curl --write-out %{http_code} --silent --output /dev/null ${SERVICE_IP})
                if [ $response = 200 ];then kubectl delete -f deploy-${curr_env}.yaml;else "deployment not successful";fi
                '''
            }
        }
     }  
}
