pipeline{
    agent any
    
    environment{
        IAMGE_NAME = "arjunckm/java-mysql-demo:${BUILD_NUMBER}"
        REGION_NAME = "us-east-1"
        NAMESPACE = "javamysql"
        CLUSTER_NAME = "javamysqldemo"
    }

    
    stages{
        stage("Branch-Selection"){
            steps{
                script{
                    def branchInput = input(
                        id = "branchSelect"
                        message: "Select the branch to build"
                        parameters: [string(name: 'BRANCH', defaultValue: 'main', description: 'Enter the branch name to build')]
                    )
                    env.BRANCH_NAME = branchInput
                    echo "✅ Selected Branch: ${env.BRANCH_NAME}"
                }
            }

        }
    }
}