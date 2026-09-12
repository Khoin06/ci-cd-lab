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
            description: 'DEPLOY_NORMAL = deploy bình thường. Chọn SHA để rollback PROD.'
        )
    }

    environment {
        IMAGE_NAME = 'ghcr.io/khoin06/ci-cd-lab'

        DEV_CONTAINER  = 'ci-cd-dev'
        UAT_CONTAINER  = 'ci-cd-uat'
        PROD_CONTAINER = 'ci-cd-prod'

        DEV_PORT  = '5001'
        UAT_PORT  = '5002'
        PROD_PORT = '5003'
    }

    stages {

        // ========================================
        // CHECKOUT
        // ========================================
        stage('Checkout') {
            steps {
                checkout scm
            }
        }


        // ========================================
        // MANUAL ROLLBACK PROD
        // ========================================
        stage('Manual Rollback PROD') {

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

                echo "================================"
                echo "MANUAL ROLLBACK PROD"
                echo "Image: ${ROLLBACK_IMAGE}"
                echo "================================"

                sh """
                    ssh \
                      -o ServerAliveInterval=30 \
                      -o ServerAliveCountMax=3 \
                      master-02@192.168.56.12 \
                      'set -e; \
                       docker pull ${ROLLBACK_IMAGE}; \
                       docker rm -f ${PROD_CONTAINER} 2>/dev/null || true; \
                       docker run -d \
                         --name ${PROD_CONTAINER} \
                         -p ${PROD_PORT}:5000 \
                         ${ROLLBACK_IMAGE}'
                """
            }
        }


        // ========================================
        // GET COMMIT SHA
        // ========================================
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


        // ========================================
        // TEST
        // ========================================
        stage('Test') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                sh '''
                    echo "===== CREATE VENV ====="

                    rm -rf venv-ci

                    python3 -m venv venv-ci

                    . venv-ci/bin/activate

                    echo "===== INSTALL DEPENDENCIES ====="

                    pip install -r requirements.txt

                    echo "===== RUN TEST ====="

                    pytest
                '''
            }
        }


        // ========================================
        // BUILD DOCKER IMAGE
        // ========================================
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


        // ========================================
        // PUSH GHCR
        // ========================================
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


        // ========================================
        // DEPLOY DEV
        // ========================================
        stage('Deploy DEV') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                echo "===== DEPLOY DEV ====="

                sh """
                    ssh master-02@192.168.56.12 \
                    'set -e; \
                     docker pull ${IMAGE}; \
                     docker rm -f ${DEV_CONTAINER} 2>/dev/null || true; \
                     docker run -d \
                       --name ${DEV_CONTAINER} \
                       -p ${DEV_PORT}:5000 \
                       ${IMAGE}'
                """
            }
        }


        // ========================================
        // HEALTH CHECK DEV
        // ========================================
        stage('Health Check DEV') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                sleep 5

                sh """
                    ssh master-02@192.168.56.12 \
                    "curl -fsS http://localhost:${DEV_PORT}/health"
                """

                echo "DEV HEALTH CHECK PASSED"
            }
        }


        // ========================================
        // DEPLOY UAT
        // ========================================
        stage('Deploy UAT') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                echo "===== DEPLOY UAT ====="

                sh """
                    ssh master-02@192.168.56.12 \
                    'set -e; \
                     docker pull ${IMAGE}; \
                     docker rm -f ${UAT_CONTAINER} 2>/dev/null || true; \
                     docker run -d \
                       --name ${UAT_CONTAINER} \
                       -p ${UAT_PORT}:5000 \
                       ${IMAGE}'
                """
            }
        }


        // ========================================
        // HEALTH CHECK UAT
        // ========================================
        stage('Health Check UAT') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                sleep 5

                sh """
                    ssh master-02@192.168.56.12 \
                    "curl -fsS http://localhost:${UAT_PORT}/health"
                """

                echo "UAT HEALTH CHECK PASSED"
            }
        }


        // ========================================
        // APPROVAL BEFORE PROD
        // ========================================
        stage('Approve Production') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                input(
                    message: "Deploy ${IMAGE} to Production?",
                    ok: "Approve"
                )
            }
        }


        // ========================================
        // SAVE CURRENT PROD VERSION
        // ========================================
        stage('Get Current PROD Version') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                script {

                    env.PREVIOUS_PROD_IMAGE = sh(
                        script: """
                            ssh master-02@192.168.56.12 \
                            "docker inspect ${PROD_CONTAINER} \
                             --format='{{.Config.Image}}' \
                             2>/dev/null || true"
                        """,
                        returnStdout: true
                    ).trim()
                }

                echo "Previous PROD image: ${PREVIOUS_PROD_IMAGE}"
            }
        }


        // ========================================
        // DEPLOY PROD
        // ========================================
        stage('Deploy PROD') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                echo "===== DEPLOY PROD ====="
                echo "Image: ${IMAGE}"

                sh """
                    ssh master-02@192.168.56.12 \
                    'set -e; \
                     docker pull ${IMAGE}; \
                     docker rm -f ${PROD_CONTAINER} 2>/dev/null || true; \
                     docker run -d \
                       --name ${PROD_CONTAINER} \
                       -p ${PROD_PORT}:5000 \
                       ${IMAGE}'
                """
            }
        }


        // ========================================
        // PROD HEALTH CHECK + AUTO ROLLBACK
        // ========================================
        stage('Health Check PROD') {

            when {
                expression {
                    return params.ROLLBACK_TAG == 'DEPLOY_NORMAL'
                }
            }

            steps {

                script {

                    echo "===== WAIT PROD START ====="

                    sleep 5

                    echo "===== PROD HEALTH CHECK ====="

                    def status = sh(
                        script: """
                            ssh master-02@192.168.56.12 \
                            "curl -fsS http://localhost:${PROD_PORT}/health"
                        """,
                        returnStatus: true
                    )


                    if (status != 0) {

                        echo "===== PROD HEALTH CHECK FAILED ====="

                        if (env.PREVIOUS_PROD_IMAGE?.trim()) {

                            echo "===== AUTO ROLLBACK PROD ====="
                            echo "Rollback to: ${PREVIOUS_PROD_IMAGE}"

                            sh """
                                ssh master-02@192.168.56.12 \
                                'set -e; \
                                 docker pull ${PREVIOUS_PROD_IMAGE}; \
                                 docker rm -f ${PROD_CONTAINER} \
                                   2>/dev/null || true; \
                                 docker run -d \
                                   --name ${PROD_CONTAINER} \
                                   -p ${PROD_PORT}:5000 \
                                   ${PREVIOUS_PROD_IMAGE}'
                            """

                            echo "===== PROD ROLLBACK COMPLETED ====="

                        } else {

                            echo "No previous PROD image found."
                        }

                        error(
                            "PROD health check failed. " +
                            "Automatic rollback executed."
                        )
                    }


                    echo "===== PROD HEALTH CHECK PASSED ====="
                }
            }
        }
    }


    // ============================================
    // POST
    // ============================================
    post {

        success {

            echo "================================"
            echo "PIPELINE SUCCESS"
            echo "================================"

            script {

                if (params.ROLLBACK_TAG == 'DEPLOY_NORMAL') {

                    echo "DEV  : http://192.168.56.12:5001"
                    echo "UAT  : http://192.168.56.12:5002"
                    echo "PROD : http://192.168.56.12:5003"

                    echo "Image: ${IMAGE}"

                } else {

                    echo "Manual rollback completed"
                    echo "Image: ${ROLLBACK_IMAGE}"
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
