pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'ec2-100-24-42-119.compute-1.amazonaws.com:8082'
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        APP_NAME = 'cicd-demo-app'
        NAMESPACE = 'production'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'npm run test:ci'
            }
            post {
                always {
                    junit 'coverage/junit.xml'
                    publishHTML([
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} .
                        docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                                   ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('Push to Nexus') {
            steps {
                script {
                    sh """
                        echo ${NEXUS_CREDENTIALS_PSW} | docker login ${DOCKER_REGISTRY} \
                            -u ${NEXUS_CREDENTIALS_USR} --password-stdin
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl set image deployment/cicd-demo-app \
                                cicd-demo-app=${DOCKER_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                                -n ${NAMESPACE}
                            kubectl rollout status deployment/cicd-demo-app -n ${NAMESPACE}
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "docker logout ${DOCKER_REGISTRY}"
            cleanWs()
        }
        success {
            echo "Pipeline succeeded! Deployed ${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}