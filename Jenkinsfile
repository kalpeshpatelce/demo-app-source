pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "kalpeshpatelce/demo-app"
        GITHUB_USERNAME  = "kalpeshpatelce"
        IMAGE_TAG        = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh 'node --version || true'
                sh 'npm test || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} -t ${DOCKER_IMAGE}:latest ."
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Update K8s Manifest') {
            steps {
                withCredentials([string(
                    credentialsId: 'github-token',
                    variable: 'GIT_TOKEN'
                )]) {
                    sh """
                        if ! command -v yq &> /dev/null; then
                            wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
                            chmod +x /usr/local/bin/yq
                        fi

                        rm -rf manifests-repo
                        git clone https://${GITHUB_USERNAME}:\${GIT_TOKEN}@github.com/${GITHUB_USERNAME}/demo-app-manifests.git manifests-repo
                        cd manifests-repo
                        yq eval ".spec.template.spec.containers[0].image = \\"${DOCKER_IMAGE}:${IMAGE_TAG}\\"" -i deployment.yaml
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins"
                        git add deployment.yaml
                        git commit -m "ci: update image tag to ${IMAGE_TAG}"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        success { echo "SUCCESS - Image pushed: ${DOCKER_IMAGE}:${IMAGE_TAG}" }
        failure { echo "FAILED - Check logs above" }
        always  { sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true" }
    }
}