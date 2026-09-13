pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '251523190381'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        AUTH_REPO = 'streamingapp-auth'
        STREAMING_REPO = 'streamingapp-streaming'
        ADMIN_REPO = 'streamingapp-admin'
        CHAT_REPO = 'streamingapp-chat'
        FRONTEND_REPO = 'streamingapp-frontend'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-jenkins']
                ]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} |
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Build Images') {
            parallel {

                stage('Build Auth') {
                    steps {
                        sh '''
                            docker build \
                              -t ${ECR_REGISTRY}/${AUTH_REPO}:${BUILD_NUMBER} \
                              -t ${ECR_REGISTRY}/${AUTH_REPO}:latest \
                              ./backend/authService
                        '''
                    }
                }

                stage('Build Streaming') {
                    steps {
                        sh '''
                            docker build \
                              -t ${ECR_REGISTRY}/${STREAMING_REPO}:${BUILD_NUMBER} \
                              -t ${ECR_REGISTRY}/${STREAMING_REPO}:latest \
                              ./backend/streamingService
                        '''
                    }
                }

                stage('Build Admin') {
                    steps {
                        sh '''
                            docker build \
                              -t ${ECR_REGISTRY}/${ADMIN_REPO}:${BUILD_NUMBER} \
                              -t ${ECR_REGISTRY}/${ADMIN_REPO}:latest \
                              ./backend/adminService
                        '''
                    }
                }

                stage('Build Chat') {
                    steps {
                        sh '''
                            docker build \
                              -t ${ECR_REGISTRY}/${CHAT_REPO}:${BUILD_NUMBER} \
                              -t ${ECR_REGISTRY}/${CHAT_REPO}:latest \
                              ./backend/chatService
                        '''
                    }
                }

                stage('Build Frontend') {
                    steps {
                        sh '''
                            docker build \
                              --build-arg REACT_APP_AUTH_API_URL=/api/auth \
                              --build-arg REACT_APP_STREAMING_API_URL=/api/streaming \
                              --build-arg REACT_APP_STREAMING_PUBLIC_URL= \
                              --build-arg REACT_APP_ADMIN_API_URL=/api/admin \
                              --build-arg REACT_APP_CHAT_API_URL=/api/chat \
                              --build-arg REACT_APP_CHAT_SOCKET_URL= \
                              -t ${ECR_REGISTRY}/${FRONTEND_REPO}:${BUILD_NUMBER} \
                              -t ${ECR_REGISTRY}/${FRONTEND_REPO}:latest \
                              ./frontend
                        '''
                    }
                }
            }
        }

        stage('Push Images') {
            parallel {

                stage('Push Auth') {
                    steps {
                        sh '''
                            docker push ${ECR_REGISTRY}/${AUTH_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${AUTH_REPO}:latest
                        '''
                    }
                }

                stage('Push Streaming') {
                    steps {
                        sh '''
                            docker push ${ECR_REGISTRY}/${STREAMING_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${STREAMING_REPO}:latest
                        '''
                    }
                }

                stage('Push Admin') {
                    steps {
                        sh '''
                            docker push ${ECR_REGISTRY}/${ADMIN_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${ADMIN_REPO}:latest
                        '''
                    }
                }

                stage('Push Chat') {
                    steps {
                        sh '''
                            docker push ${ECR_REGISTRY}/${CHAT_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${CHAT_REPO}:latest
                        '''
                    }
                }

                stage('Push Frontend') {
                    steps {
                        sh '''
                            docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:latest
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'All StreamingApp images built and pushed successfully to ECR.'
        }

        failure {
            echo 'Jenkins pipeline failed. Check the failed stage for details.'
        }

        always {
            sh 'docker logout ${ECR_REGISTRY} || true'
        }
    }
}
