pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Docker Image Tag to Deploy')
    }

    stages {
        stage('Docker Pull') {
            steps {
                script {
                    sshagent(credentials: ['ssh-deployment-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_DEPLOY_SERVER} '
                            echo 'Testing"
                        '
                        """
                    }
                }
            }
        }

        stage('Update Version') {
            steps {
                echo "Updating deployed version to ${IMAGE_TAG}"
            }
        }
    }
}
