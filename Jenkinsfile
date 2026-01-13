pipeline{
    agent any
    tools{
        jdk 'java-17'
        maven 'maven'
    }
    environment{
        IAMGE_NAME = "arjunckm/java-mysql-demo:${BUILD_NUMBER}"
        REGION_NAME = "us-east-1"
        NAMESPACE = "javamysqldemo"
        CLUSTER_NAME = "javamysqldemo-cluster"
    }

    
    stages{
        stage("Branch-Selection"){
            steps{
                script{
                    def branchInput = input(
                        id = "branchSelect",
                        message: "Select the branch to build",
                        parameters: [string(name: 'BRANCH', defaultValue: 'main', description: 'Enter the branch name to build')]
                    )
                    env.BRANCH_NAME = branchInput.BRANCH
                    echo "✅ Selected Branch: ${env.BRANCH_NAME}"
                }
            }
        }
        stage('Clone Repository') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/Gotoman12/Java-Demo-k8s.git'
            }
        }
        stage("Package"){
            steps{
                    sh 'mvn clean package'
            }
        }
        stage("Docker-Build"){
            steps{
                    sh 'docker build -t ${IMAGE} .'
            }
        }
        stage('docker-cred'){
            steps{
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker_hubcred', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Login to Docker Hub
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                }
            }
         }
      }
      stage("Docker-push"){
        steps{
            sh 'docker push ${IMAGE_NAME}'
        }
      }
      stage("KUBE_CONFIG"){
        steps{
            sh "aws eks update-kubeconfig --region ${REGION_NAME} --name ${CLUSTER_NAME}"
        }
      }
      stage("Deploy To Kubernetes"){
        steps{
             withKubeConfig(caCertificate: '', clusterName: "${CLUSTER_NAME}" , contextName: '', credentialsId: 'kube', namespace: "${NAMESPACE}",restrictKubeConfigAccess: false, serverUrl: 'https://AD29A1EDDAA2894AE3B00FA20FDD8EDB.gr7.us-east-1.eks.amazonaws.com') {
                    sh '''
                    kubectl apply -f k8s/deployment.yml -n ${NAMESPACE}
                    kubectl apply -f k8s/deployment.yml -n ${NAMESPACE}
                    kubectl apply -f k8s/mysql-secret.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/mysql-configmap.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/mysql-pvc.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/mysql-statefulset.yaml -n ${NAMESPACE}
                    '''
                }
           }
        }
        stage("Verify the Deployment"){
            steps{
                withKubeConfig(caCertificate: '', clusterName: "${CLUSTER_NAME}" , contextName: '', credentialsId: 'kube', namespace: '${NAMESPACE}',restrictKubeConfigAccess: false, serverUrl: 'https://AD29A1EDDAA2894AE3B00FA20FDD8EDB.gr7.us-east-1.eks.amazonaws.com'){
                    sh 'kubectl get pods -n ${NAMESPACE}'
                    sh 'kubectl get  svc -n ${NAMESPACE}'
                }
            }
        }
    }
}