pipeline {
    agent { label 'master-03' }

    parameters {
        choice(
            name: 'ROLLBACK_TAG',
            choices: [
                'DEPLOY_NORMAL',
                '5e6c9a8',
                '8b6495d'
            ],
            description: 'Chọn DEPLOY_NORMAL để deploy bình thường hoặc chọn SHA để rollback'
        )
    }

    environment {
        IMAGE_NAME = 'ghcr.io/khoin06/ci-cd-lab'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Rollback') {
            when {
                expression {
                    return params.ROLLBACK_TAG != 'DEPLOY_NORMAL'
                }
            }

            steps {
                script {
                    env.ROLLBACK_IMAGE =
                        "${IMAGE_NAME}:${params.ROLLBACK_TAG}"
                }

                echo "===== MANUAL ROLLBACK ====="
                echo "Rollback image: ${ROLLBACK_IMAGE}"

                sh """
                    ssh \
                      -o ServerAliveInterval=30 \
                      -o ServerAliveCountMax=3 \
                      master-02@192.168.56.12 \
                      'set -e; \
                       docker pull ${ROLLBACK_IMAGE}; \
                       docker rm -f ci-cd-app 2>/dev/null || true; \
                       docker run -d \
                         --name ci-cd-app \
                         -p 5000:5000 \
                         ${ROLLBACK_IMAGE}'
                """
            }
        }

        stage('Get Commit SHA') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                script {
                    env.GIT_SHORT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE = "${IMAGE_NAME}:${GIT_SHORT_SHA}"
                }

                echo "Commit SHA: ${GIT_SHORT_SHA}"
                echo "Docker Image: ${IMAGE}"
            }
        }

        stage('Test') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                sh '''
                    python3 -m venv venv-ci
                    . venv-ci/bin/activate
                    pip install -r requirements.txt
                    pytest
                '''
            }
        }

        stage('Build Docker Image') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                sh '''
                    docker build -t "$IMAGE" .
                '''
            }
        }

        stage('Push GHCR') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                retry(3) {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'ghcr-credentials',
                            usernameVariable: 'GHCR_USER',
                            passwordVariable: 'GHCR_TOKEN'
                        )
                    ]) {
                        sh '''
                            echo "$GHCR_TOKEN" | docker login ghcr.io \
                                -u "$GHCR_USER" \
                                --password-stdin

                            docker push "$IMAGE"
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                sh """
                    ssh \
                      -o ServerAliveInterval=30 \
                      -o ServerAliveCountMax=3 \
                      master-02@192.168.56.12 \
                      'set -e; \
                       docker pull ${IMAGE}; \
                       docker rm -f ci-cd-app 2>/dev/null || true; \
                       docker run -d \
                         --name ci-cd-app \
                         -p 5000:5000 \
                         ${IMAGE}'
                """
            }
        }
    }
}
