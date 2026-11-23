
pipeline {
    agent any

    tools {
        nodejs 'NodeJS20'
    }

    environment {
        AWS_REGION = 'ap-south-1'

        // Set this to your AWS account ID
        AWS_ACCOUNT_ID = '994878981749'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        BACKEND_IMAGE = 'nutriflow-backend'
        FRONTEND_IMAGE = 'nutriflow-frontend'

        SONAR_PROJECT_KEY = 'nutriflow'
        SONAR_HOST_URL = 'http://3.110.42.196:9000'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Image tag: ${IMAGE_TAG}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('backend') {
                    sh 'npm ci'
                }

                dir('frontend') {
                    sh 'npm ci'
                }
            }
        }

        stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool 'SonarScanner'

            withSonarQubeEnv('sonarqube') {
                sh """
                    '${scannerHome}/bin/sonar-scanner' \
                    -Dsonar.projectKey=nutriflow \
                    -Dsonar.projectName=NutriFlow \
                    -Dsonar.sources=backend,frontend/src \
                    -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**
                """
            }
        }
    }
}
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir('backend') {
                    sh 'npm test --if-present'
                }

                dir('frontend') {
                    sh 'npm test --if-present'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm run build'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_REGISTRY}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${BACKEND_IMAGE}:latest \
                      ./backend

                    docker build \
                      -t ${ECR_REGISTRY}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${FRONTEND_IMAGE}:latest \
                      ./frontend
                '''
            }
        }

        
stage('Login to ECR') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'aws-ecr-credentials',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
            sh '''
                export AWS_DEFAULT_REGION=ap-south-1

                aws ecr get-login-password --region $AWS_DEFAULT_REGION |
                docker login --username AWS --password-stdin \
                994878981749.dkr.ecr.ap-south-1.amazonaws.com
            '''
        }
    }
}


        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push ${ECR_REGISTRY}/${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${BACKEND_IMAGE}:latest

                    docker push ${ECR_REGISTRY}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${FRONTEND_IMAGE}:latest
                '''
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                sh '''
                    sed -i "s|image: .*nutriflow-backend:.*|image: ${ECR_REGISTRY}/${BACKEND_IMAGE}:${IMAGE_TAG}|" \
                      k8s/backend-deployment.yaml

                    sed -i "s|image: .*nutriflow-frontend:.*|image: ${ECR_REGISTRY}/${FRONTEND_IMAGE}:${IMAGE_TAG}|" \
                      k8s/frontend-deployment.yaml
                '''
            }
        }

        stage('Push GitOps Changes') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add k8s/

                        if git diff --cached --quiet; then
                            echo "No Kubernetes manifest changes"
                        else
                            git commit -m "Update NutriFlow images to ${IMAGE_TAG}"

                            git push https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/jijojohn911/nutriflow.git HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker image prune -f || true
            '''
        }

        success {
            echo "NutriFlow pipeline completed successfully."
            echo "Argo CD will sync the updated Kubernetes manifests."
        }

        failure {
            echo "NutriFlow pipeline failed. Check the stage logs."
        }
    }
}

