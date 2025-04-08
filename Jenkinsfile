pipeline {
    agent any
    
    environment {
        IMAGE_NAME = 'nodejs-demo-app'
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
        
        
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:latest .'
                sh 'docker tag $IMAGE_NAME:latest $IMAGE_NAME:$BUILD_NUMBER'
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
        
        stage('Deploy') {
            steps {
                echo 'Deploying to local environment...'
                sh '''
                    # Stop existing container if it exists
                    docker stop nodejs-app-container || true
                    docker rm nodejs-app-container || true
                    
                    # Run the new version
                    docker run -d -p 3000:3000 --name nodejs-app-container $IMAGE_NAME:latest
                '''
            }
        }
    }
    
    post {
        always {
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