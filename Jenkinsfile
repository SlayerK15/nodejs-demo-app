pipeline {
    agent {
        docker {
            image 'node:18-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-credentials')
        IMAGE_NAME = 'nodejs-demo-app'
        DOCKER_HUB_USERNAME = "${env.DOCKER_HUB_CREDS_USR}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Running tests...'
                // Currently no tests defined, but you can add them here
                // sh 'npm test'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:latest .'
                sh 'docker tag $IMAGE_NAME:latest $DOCKER_HUB_USERNAME/$IMAGE_NAME:latest'
                sh 'docker tag $IMAGE_NAME:latest $DOCKER_HUB_USERNAME/$IMAGE_NAME:$BUILD_NUMBER'
            }
        }
        
        stage('Test Application') {
            steps {
                sh '''
                    docker run -d -p 3000:3000 --name app-container $IMAGE_NAME:latest
                    sleep 5
                    
                    response=$(curl -s http://localhost:3000)
                    echo "Response: $response"
                    
                    if [[ "$response" == *"Hello from nodejs-demo-app"* ]]; then
                        echo "Test passed! Application is working as expected."
                    else
                        echo "Test failed! Application response doesn't match expected output."
                        exit 1
                    fi
                    
                    docker stop app-container
                    docker rm app-container
                '''
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                sh 'echo $DOCKER_HUB_CREDS_PSW | docker login -u $DOCKER_HUB_CREDS_USR --password-stdin'
                sh 'docker push $DOCKER_HUB_USERNAME/$IMAGE_NAME:latest'
                sh 'docker push $DOCKER_HUB_USERNAME/$IMAGE_NAME:$BUILD_NUMBER'
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying to production...'
            }
        }
    }
    
    post {
        always {
            sh 'docker logout'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}