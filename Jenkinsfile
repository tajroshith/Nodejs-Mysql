pipeline {
agent {
  label 'Agent-1'
}
parameters {
  choice choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green', name: 'DEPLOY_ENV'
  choice choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment', name: 'DOCKER_TAG'
  booleanParam description: 'Switch traffic between Blue and Green', name: 'SWITCH_TRAFFIC'
}
environment {
  SCANNER_HOME= tool 'sonar-scanner' //Single Quotes for literal string.
  IMAGE_NAME= 'zookl0/nodejs-mysql'  //Single Quotes for literal string.
  KUBE_NAMESPACE = 'webapps'  //Single Quotes for literal string.
  TAG= "${params.DOCKER_TAG}"  //Double Quotes for string interpolation.
}
stages{
    stage('GitCheckOut'){
        steps{
            git branch: 'main', credentialsId: 'Github_Creds', url: 'https://github.com/tajroshith/Nodejs-Mysql.git'
        }
    }
    stage('SonarQube-Analysis'){
        steps{
         withSonarQubeEnv('sonar-server') {
            sh """
            $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Nodejs-Mysql -Dsonar.projectName=Nodejs-Mysql
            """
           }
        }
    }
    stage('Wait-for-Quality-Gate'){
        steps{
            timeout(5) {
                waitForQualityGate abortPipeline: false
            }
        }
    }
    stage('Trivy-fs-scan'){
        steps{
          sh "trivy fs --format table -o trivy-fs-report.html ."
        }
    }
    stage('DockerBuild'){
        steps{
           withCredentials([string(credentialsId: 'Docker_hub_pwd', variable: 'Docker_Hub_Pwd')]) {
            sh "docker login -u zookl0 -p${Docker_Hub_Pwd}"
           }
            sh "docker build -t $IMAGE_NAME:$TAG -f Dockerfile-custom ."
        }
    }
    stage('Trivy-image-scan'){
        steps{
          sh "trivy image --format table -o trivy-image-report.html $IMAGE_NAME:$TAG"
        }
    }
    stage('DockerPush'){
        steps{
           withCredentials([string(credentialsId: 'Docker_hub_pwd', variable: 'Docker_Hub_Pwd')]) {
            sh "docker login -u zookl0 -p${Docker_Hub_Pwd}"
           }
            sh "docker push $IMAGE_NAME:$TAG"
        }
    }
    stage('Deploy to k8s - app-service'){
        steps{
          withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EDA9319A9C3B7BDE607CD28A639493C4.gr7.ap-south-1.eks.amazonaws.com') {
            sh "kubectl get svc app-service -n $KUBE_NAMESPACE || kubectl apply -f app-service.yml -n $KUBE_NAMESPACE"
           }
        }
    }
    stage('Deploy to k8s ALL'){
        steps{
             script {
              def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
              withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EDA9319A9C3B7BDE607CD28A639493C4.gr7.ap-south-1.eks.amazonaws.com') {
                 sh "kubectl apply -f secret.yml -n $KUBE_NAMESPACE"
                 sh "kubectl apply -f configmap.yml -n $KUBE_NAMESPACE"                 
                 sh "kubectl apply -f pv.yml -n $KUBE_NAMESPACE"
                 sh "kubectl apply -f pvc.yml -n $KUBE_NAMESPACE"
                 sh "kubectl apply -f mysql-deployment.yml -n $KUBE_NAMESPACE"
                 sh "kubectl apply -f ${deploymentFile} -n $KUBE_NAMESPACE"
              }
           }
        }
    }
    stage('Switch Traffic'){
        when { 
          expression { params.SWITCH_TRAFFIC }
        }
        steps{
           withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EDA9319A9C3B7BDE607CD28A639493C4.gr7.ap-south-1.eks.amazonaws.com') {
              sh "kubectl set selector service app-service app=app,version=${params.DEPLOY_ENV} -n ${KUBE_NAMESPACE}"
              echo "Traffic switched to ${params.DEPLOY_ENV} environment."
            }
        }
    }
    stage('Verify Deployments'){
        steps{
          withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://EDA9319A9C3B7BDE607CD28A639493C4.gr7.ap-south-1.eks.amazonaws.com') {
            sh "kubectl get pods -l version=${params.DEPLOY_ENV} -n ${KUBE_NAMESPACE}"
            sh "kubectl get svc app-service -n ${KUBE_NAMESPACE}"
           }
        }
    }
}//Stages Closing
}//Pipeline Closing


/* NOTES:-
In the ternary operator(?:) params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml', 
if params.DEPLOY_ENV equals 'blue', the value after the ? ('app-deployment-blue.yml') is assigned to deploymentFile.
If params.DEPLOY_ENV is not 'blue', the value after the : ('app-deployment-green.yml') is used instead.
*/