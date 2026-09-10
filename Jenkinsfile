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
                    python3 -m venv venv-ci
                    . venv-ci/bin/activate
                    pip install -r requirements.txt
                    pytest
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $IMAGE .
                '''
            }
        }

        stage('Push GHCR') {
            steps {
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

        stage('Deploy') {
            steps {
                sh '''
                    ssh master-02@192.168.56.12 '
                        docker pull ${IMAGE}

                        docker rm -f ci-cd-app 2>/dev/null || true

                        docker run -d \
                            --name ci-cd-app \
                            -p 5000:5000 \
                            ${IMAGE}
                    '
                '''
            }
        }
    }
}
