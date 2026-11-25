
pipeline {
    agent any

    environment {
        HARBOR_URL  = '10.25.118.93'
        HARBOR_PROJECT = 'hello-jenkins'
        IMAGE_NAME  = 'hello-jenkins'
        IMAGE_TAG   = "${BUILD_NUMBER}"
        FULL_IMAGE  = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest -v
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${FULL_IMAGE} .
                '''
            }
        }

        stage('Push to Harbor') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-credentials',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$HARBOR_PASSWORD" | docker login ${HARBOR_URL} \
                            -u "$HARBOR_USER" --password-stdin

                        docker push ${FULL_IMAGE}
                        docker logout ${HARBOR_URL}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        kubectl set image deployment/hello-jenkins \
                            hello-jenkins=${FULL_IMAGE} \
                            -n demo

                        kubectl rollout status deployment/hello-jenkins \
                            -n demo --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD successful! Deployed ${FULL_IMAGE}"
        }
        failure {
            echo "CI/CD pipeline failed. Check the stage logs."
        }
    }
}
