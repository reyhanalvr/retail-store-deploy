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
                expression { hasUiChanges }
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

        stage('Get Latest Tag and Bump Version') {
            when {
                expression { hasUiChanges }
            }
            steps {
                script {
                    // Mengambil tag terbaru dari Git, jika tidak ada, default ke 0.0.1
                    def latestTag = sh(script: "git describe --tags --abbrev=0 || echo '0.0.0'", returnStdout: true).trim()
                    echo "Latest tag: ${latestTag}"
        
                    // Pisahkan versi
                    def versionParts = latestTag.tokenize('.')
                    if (versionParts.size() < 3) {
                        // Jika tidak ada tag yang valid, set ke versi awal
                        versionParts = ['0', '0', '1'] // Set tag awal ke 0.0.1
                    } else {
                        // Menaikkan patch version
                        versionParts[2] = (versionParts[2].toInteger() + 1).toString()
                    }
                    def newTag = versionParts.join('.')
                    echo "New tag: ${newTag}"
        
                    // Set IMAGE_TAG untuk digunakan dalam tahap berikutnya
                    env.IMAGE_TAG = newTag
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-build-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                            cd /home/alvaro/retail-store-deploy/retail-store-sample-app/src/ui &&
                            docker buildx build --no-cache -t registry.alvaro.studentdumbways.my.id/retail-store-sample/ui:${IMAGE_TAG} .
                        '
                        """
                    }
                }
            }
        }

        stage('Basic Security Checks') {
            when {
                expression { hasUiChanges }
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
                expression { hasUiChanges }
            }
            steps {
                sshagent(credentials: ['ssh-build-server']) {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        script {
                            sh """
                            ssh -o StrictHostKeyChecking=no ${SSH_BUILD_SERVER} '
                                echo ${DOCKER_PASS} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} --password-stdin &&
                                docker push registry.alvaro.studentdumbways.my.id/retail-store-sample/ui:${IMAGE_TAG}
                            '
                            """
                        }
                    }
                }
            }
        }

        stage('Update Values.yaml with New Tag') {
            when {
                expression { hasUiChanges }
            }
            steps {
                script {
                    sh """
                    sed -i "s/tag: .*/tag: ${IMAGE_TAG}/" retail-store-sample-app/deploy/kubernetes/charts/ui/values.yaml
                    """
                }
            }
        }

        stage('Commit and Push New Image Changes') {
            when {
                expression { hasUiChanges }
            }
            steps {
                script {
                    sh """
                    git add retail-store-sample-app/deploy/kubernetes/charts/ui/values.yaml
                    git commit -m "Update image tag to ${IMAGE_TAG} for deployment"
                    git push origin master
                    """
                }
            }
        }

        stage('Deploy to Argo CD') {
            when {
                expression { hasUiChanges }
            }
            steps {
                script {
                    sshagent(credentials: ['ssh-server']) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_DEPLOY_SERVER} '
                            argocd app sync retail-store-ui --grpc-web
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
