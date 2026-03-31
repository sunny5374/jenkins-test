pipeline {
    agent any

    environment {
        REGISTRY = "amdp-registry.skala-ai.com"
        PROJECT = "skala26a-ai2"
        IMAGE_NAME = "jenkins-day1"
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE = "${REGISTRY}/${PROJECT}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '소스코드 가져오기'
                checkout scm
            }
        }

        stage('Code Build') {
            steps {
                echo '코드 빌드(검증) 수행'
                sh '''
                    pwd
                    ls -al
                    test -f Dockerfile
                    echo "Code build/check done"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                echo '도커 이미지 빌드'
                sh '''
                    docker build -t ${FULL_IMAGE} .
                '''
            }
        }

        stage('Docker Login') {
            steps {
                echo 'Harbor 로그인'
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-cred',
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        echo "${HARBOR_PASS}" | docker login amdp-registry.skala-ai.com -u "${HARBOR_USER}" --password-stdin
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Harbor Registry로 이미지 Push'
                sh '''
                    docker push ${FULL_IMAGE}
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${FULL_IMAGE}"
        }
        failure {
            echo "FAILED"
        }
        always {
            sh 'docker logout amdp-registry.skala-ai.com || true'
        }
    }
}
