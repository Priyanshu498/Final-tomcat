pipeline {
    agent any
    environment {
        TERRAFORM_WORKSPACE = "/var/lib/jenkins/workspace/tool_deploy/tomcat-infra/"
        INSTALL_WORKSPACE = "/var/lib/jenkins/workspace/tool_deploy/tomcat/"
        PATH = "/usr/local/bin:${env.PATH}"
    }
    parameters {
        choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'Select action: apply or destroy')
    }
    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/Priyanshu498/Final-tomcat.git'
            }
        }
        stage('Terraform Init') {
            steps {
                script {
                    try {
                        sh "cd ${env.TERRAFORM_WORKSPACE} && terraform init"
                    } catch (Exception e) {
                        error "Terraform Init failed: ${e}"
                    }
                }
            }
        }

        stage('Terraform Plan') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    try {
                        sh "cd ${env.TERRAFORM_WORKSPACE} && terraform plan"
                    } catch (Exception e) {
                        error "Terraform Plan failed: ${e}"
                    }
                }
            }
        }

        stage('Approval For Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                input "Do you want to apply Terraform changes?"
            }
        }

        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' && currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    try {
                        sh """
                            cd ${env.TERRAFORM_WORKSPACE}
                            terraform apply -auto-approve
                            sudo cp ${env.TERRAFORM_WORKSPACE}/tom-1-key.pem ${env.INSTALL_WORKSPACE}
                            sudo chown jenkins:jenkins ${env.INSTALL_WORKSPACE}/tom-1-key.pem
                            sudo chmod 400 ${env.INSTALL_WORKSPACE}/tom-1-key.pem
                        """
                    } catch (Exception e) {
                        error "Terraform Apply failed: ${e}"
                    }
                }
            }
        }

        stage('Tool Deploy') {
            when {
                expression { params.ACTION == 'apply' && currentBuild.result == 'SUCCESS' }
            }
            steps {
                sshagent(['tom-1-key.pem']) {
                    script {
                        sh '''
                            ansible-playbook -i ./tomcat-Role/tomcat/aws_ec2.yml ./tomcat-Role/tomcat/playbook.yml
                        '''
                    }
                }
            }

        stage('Approval for Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                input "Do you want to Terraform Destroy?"
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' && currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    try {
                        sh "cd ${env.TERRAFORM_WORKSPACE} && terraform destroy -auto-approve"
                    } catch (Exception e) {
                        error "Terraform Destroy failed: ${e}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up temporary files and states...'
        }
        success {
            echo 'Succeeded!'
        }
        failure {
            echo 'Failed!'
        }
    }
}
