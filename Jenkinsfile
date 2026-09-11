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
            description: 'Chọn DEPLOY_NORMAL để deploy bình thường hoặc chọn SHA để rollback thủ công'
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

        // =========================
        // MANUAL ROLLBACK
        // =========================
        stage('Manual Rollback') {
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

        // =========================
        // GET COMMIT SHA
        // =========================
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

                    env.IMAGE =
                        "${IMAGE_NAME}:${GIT_SHORT_SHA}"
                }

                echo "Commit SHA: ${GIT_SHORT_SHA}"
                echo "Docker Image: ${IMAGE}"
            }
        }

        // =========================
        // TEST
        // =========================
        stage('Test') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

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

        // =========================
        // BUILD DOCKER
        // =========================
        stage('Build Docker Image') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                sh '''
                    echo "===== BUILD DOCKER IMAGE ====="

                    docker build -t "$IMAGE" .
                '''
            }
        }

        // =========================
        // PUSH GHCR
        // =========================
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

        // =========================
        // SAVE CURRENT VERSION
        // =========================
        stage('Get Current Version') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                script {
                    env.PREVIOUS_IMAGE = sh(
                        script: """
                            ssh \
                              -o ServerAliveInterval=30 \
                              -o ServerAliveCountMax=3 \
                              master-02@192.168.56.12 \
                              "docker inspect ci-cd-app \
                               --format='{{.Config.Image}}' \
                               2>/dev/null || true"
                        """,
                        returnStdout: true
                    ).trim()
                }

                echo "===== CURRENT VERSION ====="
                echo "Previous image: ${PREVIOUS_IMAGE}"
            }
        }

        // =========================
        // DEPLOY NEW VERSION
        // =========================
        stage('Deploy') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                echo "===== DEPLOY NEW VERSION ====="
                echo "Deploy image: ${IMAGE}"

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

        // =========================
        // HEALTH CHECK + AUTO ROLLBACK
        // =========================
        stage('Health Check') {
            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {
                script {

                    echo "===== WAIT APPLICATION START ====="

                    sleep 5

                    echo "===== HEALTH CHECK ====="

                    def status = sh(
                        script: """
                            ssh \
                              -o ServerAliveInterval=30 \
                              -o ServerAliveCountMax=3 \
                              master-02@192.168.56.12 \
                              "curl -fsS http://localhost:5000/health"
                        """,
                        returnStatus: true
                    )

                    if (status != 0) {

                        echo "===== HEALTH CHECK FAILED ====="

                        if (env.PREVIOUS_IMAGE?.trim()) {

                            echo "===== AUTO ROLLBACK ====="
                            echo "Rollback to: ${PREVIOUS_IMAGE}"

                            sh """
                                ssh \
                                  -o ServerAliveInterval=30 \
                                  -o ServerAliveCountMax=3 \
                                  master-02@192.168.56.12 \
                                  'set -e; \
                                   docker pull ${PREVIOUS_IMAGE}; \
                                   docker rm -f ci-cd-app 2>/dev/null || true; \
                                   docker run -d \
                                     --name ci-cd-app \
                                     -p 5000:5000 \
                                     ${PREVIOUS_IMAGE}'
                            """

                            echo "===== ROLLBACK COMPLETED ====="

                        } else {

                            echo "No previous image found. Cannot rollback."
                        }

                        error(
                            "New deployment failed health check. " +
                            "Automatic rollback executed."
                        )
                    }

                    echo "===== HEALTH CHECK PASSED ====="
                }
            }
        }
    }

    post {

        success {
            echo "================================"
            echo "PIPELINE SUCCESS"
            echo "================================"

            script {
                if (params.ROLLBACK_TAG == 'DEPLOY_NORMAL') {
                    echo "Deployed image: ${IMAGE}"
                } else {
                    echo "Manual rollback image: ${ROLLBACK_IMAGE}"
                }
            }
        }

        failure {
            echo "================================"
            echo "PIPELINE FAILED"
            echo "Check Console Output"
            echo "================================"
        }
    }
}
