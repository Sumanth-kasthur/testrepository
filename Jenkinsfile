pipeline {
    agent {
        label 'builds'
    }
    tools{
        dockerTool 'myDocker'
        terraform 'terraform-1.4.6'
    }
    parameters{
        booleanParam(name: 'PLAN', defaultValue: false, description: 'The infra will be planned.')
        booleanParam(name: 'APPLY', defaultValue: false, description: 'The infra will be applied to the chosen workspace.')
    }
    environment {
        REGION = 'us-west-2'
        ACCOUNT_NAME = credentials('account_name')
        DOCKER_HOST = credentials("/${env.ACCOUNT_NAME}/platform/jenkins/docker_daemon_server")
        ACCOUNT_ID = credentials('account_id')
        TF_HOME = tool('terraform-1.4.6')
        VERSION = GIT_COMMIT.take(7)
    }
    stages {
        stage('Download Dependencies'){
            steps{
                dir('airflow'){
                    downloadDependencies(
                        directory: ".dependencies",
                        repository:"gh-org-data-platform/terraform-aws-gh-dp-glue",
                        account:"${env.ACCOUNT_NAME}",
                        branch: "v1.41.0"
                    )
                }
            }
        }
        stage("Initialize & Validate"){
            steps{
                withAWSParameterStore(credentialsId: 'GH_AWS_ACCOUNT', naming: 'relative', path: '/gh', recursive: true, regionName: "${env.REGION}"){
                    sh "terraform init -input=false -backend-config=state-lock-config-${env.ACCOUNT_NAME}.properties"
                }
                script {
                    try {
                        sh "terraform workspace select ${env.ACCOUNT_NAME}"
                    } catch (err) {
                        sh "terraform workspace new ${env.ACCOUNT_NAME}"
                        sh "terraform workspace select ${env.ACCOUNT_NAME}"
                    }
                }
                echo "${env.ACCOUNT_NAME} workspace selected"
                sh "terraform validate"
            }
        }
        stage("Plan") {
            when {expression { params.PLAN == true }}
            steps {
                sh "terraform plan -out=${env.ACCOUNT_NAME}_tfplan -var 'account=${env.ACCOUNT_NAME}' -var 'commit_id=${env.VERSION}'"
            }
        }
        stage("Docker Build") {
            steps {
                dir('code/lambda') {
                    dockerBuild(
                            account_id: "${env.ACCOUNT_ID}",
                            branch: "${env.BRANCH_NAME}",

                            account: "${env.ACCOUNT_NAME}",
                            env: "${env.ACCOUNT_NAME}",

                            image_repo: "dp-tools-repositories-airflow",
                            version: "${env.VERSION}",
                            commit_id: "${env.VERSION}"
                    )
                }
            }
        }
        stage("Plan and Apply") {
            when {expression { params.APPLY == true }}
            steps {
                sh "terraform plan -out=${env.ACCOUNT_NAME}_tfplan -var 'account=${env.ACCOUNT_NAME}' -var 'commit_id=${env.VERSION}'"
                sh "terraform apply ${env.ACCOUNT_NAME}_tfplan"
            }
        }
    }
    post {
        always {
            pushLogsAndMetadata()        
        }
    }
}
