pipeline {
    agent any
    
    environment {
        IMAGE_NAME = 'nodejs-demo-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Verify Environment') {
            steps {
                sh 'node --version'
                sh 'npm --version'
                sh 'docker --version'
            }
        }
        
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
                sh 'docker tag $IMAGE_NAME:latest $IMAGE_NAME:$IMAGE_TAG'
            }
        }
        
        stage('Test Application') {
            steps {
                sh '''#!/bin/bash
                    docker run -d -p 3000:3000 --name app-container $IMAGE_NAME:latest
                    sleep 5
                    
                    response=$(curl -s http://localhost:3000)
                    echo "Response: $response"
                    
                    # Use grep instead of [[ ... ]] syntax for wider compatibility
                    if echo "$response" | grep -q "Hello from nodejs-demo-app"; then
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
        
        stage('Archive Artifacts') {
            steps {
                // Archive the application files for potential use in deployment
                archiveArtifacts artifacts: '**', excludes: 'node_modules/**'
                echo "Docker image $IMAGE_NAME:$IMAGE_TAG is ready for deployment"
            }
        }
        
        stage('Deploy Locally') {
            steps {
                sh '''#!/bin/bash
                    # Stop any existing container with the same name
                    docker stop nodejs-demo-app || true
                    docker rm nodejs-demo-app || true
                    
                    # Deploy the application locally
                    docker run -d -p 3000:3000 --name nodejs-demo-app $IMAGE_NAME:latest
                    
                    # Verify deployment
                    sleep 3
                    curl -s http://localhost:3000
                    
                    echo "Application successfully deployed locally and available at http://localhost:3000"
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