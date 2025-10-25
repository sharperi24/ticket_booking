pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME = 'sherry245'
        BACKEND_IMAGE = "${DOCKERHUB_USERNAME}/tickethub-backend"
        FRONTEND_IMAGE = "${DOCKERHUB_USERNAME}/tickethub-frontend"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from Git...'
                checkout scm
            }
        }
        
        stage('Build Backend') {
            steps {
                echo 'Building backend Docker image...'
                bat "docker build -f Dockerfile.backend -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest ."
            }
        }
        
        stage('Build Frontend') {
            steps {
                echo 'Building frontend Docker image...'
                bat "docker build -f Dockerfile.frontend -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest ."
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                bat """
                    docker rm -f test-backend 2>nul || echo Container not found
                    docker run --rm -d --name test-backend -p 5555:5000 ${BACKEND_IMAGE}:${IMAGE_TAG}
                    ping 127.0.0.1 -n 15 > nul
                    curl -f http://localhost:5555/api/health || exit /b 1
                    docker stop test-backend
                """
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing images to Docker Hub...'
                bat """
                    echo %DOCKERHUB_CREDENTIALS_PSW% | docker login -u %DOCKERHUB_CREDENTIALS_USR% --password-stdin
                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo 'Deploying to Kubernetes...'
                bat """
                    kubectl apply -f kubernetes/backend-deployment.yaml
                    kubectl apply -f kubernetes/backend-service.yaml
                    kubectl apply -f kubernetes/frontend-deployment.yaml
                    kubectl apply -f kubernetes/frontend-service.yaml
                    kubectl set image deployment/backend-deployment backend=${BACKEND_IMAGE}:${IMAGE_TAG}
                    kubectl set image deployment/frontend-deployment frontend=${FRONTEND_IMAGE}:${IMAGE_TAG}
                    kubectl rollout status deployment/backend-deployment
                    kubectl rollout status deployment/frontend-deployment
                """
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            bat '''
                docker logout
                docker rm -f test-backend 2>nul || echo Cleanup complete
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}