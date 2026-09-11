pipeline {
    agent { label 'master-03' }

    environment {
        IMAGE_NAME = 'ghcr.io/khoin06/ci-cd-lab'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Get Commit SHA') {
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
            steps {
                sh '''
                    echo "===== CREATE VENV ====="

                    python3 -m venv venv-ci

                    . venv-ci/bin/activate

                    echo "===== INSTALL DEPENDENCIES ====="

                    pip install -r requirements.txt

                    echo "===== RUN TEST ====="

                    pytest
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "===== BUILD DOCKER IMAGE ====="

                    docker build -t "$IMAGE" .
                '''
            }
        }

        stage('Push GHCR') {
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
                            echo "===== LOGIN GHCR ====="

                            echo "$GHCR_TOKEN" | docker login ghcr.io \
                                -u "$GHCR_USER" \
                                --password-stdin

                            echo "===== PUSH IMAGE ====="

                            docker push "$IMAGE"
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    echo "===== DEPLOY TO MASTER-02 ====="

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

    post {
        success {
            echo "===== PIPELINE SUCCESS ====="
            echo "Deployed image: ${IMAGE}"
        }

        failure {
            echo "===== PIPELINE FAILED ====="
        }
    }
}
