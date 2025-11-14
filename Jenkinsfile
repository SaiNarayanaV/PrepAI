pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USERNAME = credentials('docker-username')
        DOCKER_PASSWORD = credentials('docker-password')
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_USERNAME}/quizz-backend"
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/${DOCKER_USERNAME}/quizz-frontend"
        KUBECONFIG = credentials('kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '🔄 Checking out code...'
                checkout scm
            }
        }
        
        stage('Build Backend') {
            steps {
                echo '🔨 Building backend Docker image...'
                script {
                    sh '''
                        cd backend
                        docker build -t ${BACKEND_IMAGE}:${BUILD_NUMBER} .
                        docker tag ${BACKEND_IMAGE}:${BUILD_NUMBER} ${BACKEND_IMAGE}:latest
                    '''
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                echo '🔨 Building frontend Docker image...'
                script {
                    sh '''
                        cd frontend/interview-prep-ai
                        docker build -t ${FRONTEND_IMAGE}:${BUILD_NUMBER} .
                        docker tag ${FRONTEND_IMAGE}:${BUILD_NUMBER} ${FRONTEND_IMAGE}:latest
                    '''
                }
            }
        }
        
        stage('Test Backend') {
            steps {
                echo '✅ Running backend tests...'
                script {
                    sh '''
                        cd backend
                        npm install
                        # Add your test command here
                        # npm test
                    '''
                }
            }
        }
        
        stage('Push Images') {
            when {
                branch 'main'
            }
            steps {
                echo '📤 Pushing Docker images to registry...'
                script {
                    sh '''
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin ${DOCKER_REGISTRY}
                        docker push ${BACKEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${BUILD_NUMBER}
                        docker push ${FRONTEND_IMAGE}:latest
                        docker logout ${DOCKER_REGISTRY}
                    '''
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            when {
                branch 'main'
            }
            steps {
                echo '🚀 Deploying to Kubernetes...'
                script {
                    sh '''
                        export KUBECONFIG=${KUBECONFIG}
                        kubectl set image deployment/backend backend=${BACKEND_IMAGE}:${BUILD_NUMBER} -n quizz
                        kubectl set image deployment/frontend frontend=${FRONTEND_IMAGE}:${BUILD_NUMBER} -n quizz
                        kubectl rollout status deployment/backend -n quizz
                        kubectl rollout status deployment/frontend -n quizz
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                echo '🏥 Running health checks...'
                script {
                    sh '''
                        echo "Waiting for pods to be ready..."
                        kubectl wait --for=condition=ready pod -l app=backend -n quizz --timeout=300s || true
                        kubectl wait --for=condition=ready pod -l app=frontend -n quizz --timeout=300s || true
                        echo "Backend pods:"
                        kubectl get pods -n quizz -l app=backend
                        echo "Frontend pods:"
                        kubectl get pods -n quizz -l app=frontend
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            echo '🧹 Cleaning up...'
            cleanWs()
        }
    }
}
