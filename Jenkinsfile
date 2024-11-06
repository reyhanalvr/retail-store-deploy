pipeline {
    agent any

    environment {
        IMAGE_TAG = "${IMAGE}:latest"
        DIR = "/home/alvaro/retail-store-deploy"
    }

    stages {
        stage('Check for UI Changes') {
            steps {
                script {
                    def previousCommit = sh(script: 'git rev-parse HEAD^', returnStdout: true).trim()
                    def changes = sh(
                        script: "git diff --name-only ${previousCommit} HEAD | grep 'src/ui/' || true",
                        returnStdout: true
                    ).trim()
                    hasUiChanges = changes ? true : false
                    echo "Changes detected in src/ui: ${hasUiChanges}"
                }
            }
        }

        stage('Git Pull on App Servers') {
            when {
                expression { return hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-build-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                            cd /home/alvaro/retail-store-deploy &&
                            git fetch origin &&
                            git checkout master &&
                            git pull origin master
                        '
                        """
                    }
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { return hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-build-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                            cd /home/alvaro/retail-store-deploy/retail-store-sample-app/src/ui &&
                            docker buildx build -t ${IMAGE_TAG} src/ui
                        '
                        """
                    }
                }
            }
        }

        stage('Basic Security Checks') {
            when {
                expression { return hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-build-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                            echo "Displaying Docker Image Layers..." &&
                            docker history ${IMAGE_TAG} &&
                            docker inspect ${IMAGE_TAG} | grep -E "User|ExposedPorts|Env" &&
                            CONTAINER_ID=\$(docker create ${IMAGE_TAG}) &&
                            docker export \${CONTAINER_ID} | tar -tvf - | grep -E "libarchive|openssl|curl" &&
                            docker rm \${CONTAINER_ID}
                        '
                        """
                    }
                }
            }
        }

        stage('Docker Registry Login and Push') {
            when {
                expression { return hasUiChanges }
            }
            steps {
                sshagent(credentials: ['ssh-build-server']) {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        script {
                            sh """
                            ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                                echo ${DOCKER_PASS} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} --password-stdin &&
                                docker push ${IMAGE_TAG}
                            '
                            """
                        }
                    }
                }
            }
        }

        stage('Push Docker Image to Registry') {
            when {
                expression { return hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-build-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                            docker push ${IMAGE_TAG}
                        '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
