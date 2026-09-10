pipeline {
    agent { label 'master-03' }

    environment {
        IMAGE = 'ghcr.io/khoin06/ci-cd-lab:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                        docker pull ghcr.io/khoin06/ci-cd-lab:latest

                        docker rm -f ci-cd-app 2>/dev/null || true

                        docker run -d \
                            --name ci-cd-app \
                            -p 5000:5000 \
                            ghcr.io/khoin06/ci-cd-lab:latest
                    '
                '''
            }
        }
    }
}
