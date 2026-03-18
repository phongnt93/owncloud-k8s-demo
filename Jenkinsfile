pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = 'nguyenphong8852/owncloud-k8s-demo'
        IMAGE_TAG         = "${BUILD_NUMBER}"

        # Repo GitOps chứa manifest mà ArgoCD đang watch
        GITOPS_REPO       = 'github.com:phongnt93/owncloud-k8s-demo.git'
        GITOPS_BRANCH     = 'main'
        MANIFEST_FILE     = 'k8s/owncloud.yaml'
    }

    stages {
        stage('Checkout App') {
            steps {
                checkout scm
                script {
                    echo "Build image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker version
                  docker build -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} .
                  docker tag  ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_IMAGE_NAME}:latest
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                      docker push ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                      docker push ${DOCKER_IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Update GitOps Manifests') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                      rm -rf gitops-repo
                      git clone https://${GIT_USER}:${GIT_PASS}@${GITOPS_REPO} gitops-repo
                      cd gitops-repo

                      git config user.email "jenkins@local"
                      git config user.name "Jenkins"

                      echo "=== Update image tag in ${MANIFEST_FILE} ==="
                      sed -i "s|image: ${DOCKER_IMAGE_NAME}:.*|image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}|g" ${MANIFEST_FILE}
                      grep "image:" ${MANIFEST_FILE}

                      git add ${MANIFEST_FILE}
                      git commit -m "[Jenkins] Update ownCloud image to ${IMAGE_TAG}" || echo "No changes"
                      git push origin ${GITOPS_BRANCH}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI done: image pushed, GitOps repo updated. ArgoCD sẽ tự sync."
        }
    }
}

