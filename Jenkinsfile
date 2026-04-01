pipeline {
    agent any

    environment {
        GIT_REPO           = 'https://github.com/sunny5374/jenkins-test.git'
        GIT_BRANCH         = 'main'

        APP_NAME           = 'jenkins-day1'
        HARBOR_REGISTRY    = 'amdp-registry.skala-ai.com'
        HARBOR_PROJECT     = 'skala26a-ai2'
        IMAGE_REPO         = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${APP_NAME}"

        AWS_REGION         = 'ap-northeast-2'
        EKS_CLUSTER_NAME   = 'skala-2025'
        K8S_NAMESPACE      = 'class-2'

        GIT_CREDENTIALS    = 'github_personal_acess_token'
        HARBOR_CREDENTIALS = 'harbor-cred'
        AWS_CREDENTIALS    = 'aws-access-key-cred'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: "${GIT_REPO}"
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG  = "${BUILD_NUMBER}-${GIT_COMMIT_SHORT}"
                    env.FULL_IMAGE = "${IMAGE_REPO}:${IMAGE_TAG}"

                    echo "IMAGE_TAG  = ${env.IMAGE_TAG}"
                    echo "FULL_IMAGE = ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${FULL_IMAGE} .
                '''
            }
        }

        stage('Push Image to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${HARBOR_CREDENTIALS}",
                    usernameVariable: 'HARBOR_USER',
                    passwordVariable: 'HARBOR_PASS'
                )]) {
                    sh '''
                        echo "${HARBOR_PASS}" | docker login ${HARBOR_REGISTRY} -u "${HARBOR_USER}" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout ${HARBOR_REGISTRY}
                    '''
                }
            }
        }

        stage('Configure AWS CLI for EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AWS_CREDENTIALS}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        aws sts get-caller-identity

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}

                        kubectl config current-context
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    cp deploy/deployment.yaml deploy/deployment-rendered.yaml
                    sed "s|__IMAGE__|${FULL_IMAGE}|g" deploy/deployment.yaml > deploy/deployment-rendered.yaml

                    kubectl apply -n ${K8S_NAMESPACE} -f deploy/deployment-rendered.yaml
                    kubectl apply -n ${K8S_NAMESPACE} -f deploy/service.yaml

                    kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=300s
                    kubectl get pods -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo "Deployment succeeded: ${env.FULL_IMAGE}"
        }
        failure {
            echo "Deployment failed"
        }
    }
}
