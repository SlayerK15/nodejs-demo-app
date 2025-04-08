# Nodejs-Demo-App

A simple Node.js application containerized with Docker and automated with GitHub Actions and Jenkins.

## What the Application Does

This is a minimal Node.js HTTP server that:
- Listens on port 3000 (configurable via environment variables)
- Responds to all HTTP requests with the message: "Hello from nodejs-demo-app - This app has no functionality"
- Uses only Node.js core modules (no external dependencies)
- Serves as a demonstration template for containerization and CI/CD practices

## Dockerfile Explained

The Dockerfile defines how the application is containerized:

```dockerfile
# Use Node.js official image as the base image
FROM node:18-alpine

# Set working directory inside the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json (if available)
COPY package*.json ./

# Install dependencies
# Note: Since this is a simple app with no dependencies, this step 
# is included for completeness but won't install anything
RUN npm install

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
EXPOSE 3000

# Command to run the application
CMD ["npm", "start"]
```

This Dockerfile:
- Uses a lightweight Node.js 18 Alpine image
- Sets up a dedicated working directory
- Copies project files and handles dependencies
- Exposes port 3000 for external connections
- Uses npm start to launch the application

## GitHub Actions Workflow (Added April 7, 2025)

The `.github/workflows/docker-build-push.yml` file automates the build, test, and deployment process:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main  # or 'master' if that's your default branch

env:
  IMAGE_NAME: nodejs-demo-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true
          tags: ${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Start container for testing
        run: |
          docker run -d -p 3000:3000 --name app-container ${{ env.IMAGE_NAME }}:latest
          # Wait for container to fully start
          sleep 5
      
      - name: Test with curl
        run: |
          response=$(curl -s http://localhost:3000)
          echo "Response: $response"
          
          # Check if the response contains expected text
          if [[ "$response" == *"Hello from nodejs-demo-app"* ]]; then
            echo "Test passed! Application is working as expected."
          else
            echo "Test failed! Application response doesn't match expected output."
            exit 1
          fi
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}
      
      - name: Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:latest,${{ secrets.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

This workflow:
1. Triggers automatically when code is pushed to the main branch
2. Builds a Docker image from our Dockerfile
3. Runs the container on the GitHub Actions runner
4. Tests the application using curl
5. Pushes the verified image to Docker Hub if tests pass

## Jenkins Integration with GitHub Webhook (Added April 8, 2025)

The project now includes a Jenkins pipeline that is automatically triggered by GitHub webhooks:

```groovy
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
```

When a change is pushed to the GitHub repository, the webhook automatically triggers the Jenkins pipeline, which:
1. Verifies the node, npm, and docker environment
2. Checks out the code from the repository
3. Installs any dependencies
4. Builds a Docker image with both latest and build-number tags
5. Tests the application by running it in a container and verifying the response
6. Archives all application files (excluding node_modules)
7. Deploys the application locally and verifies the deployment
8. Cleans up the workspace after completion

The webhook integration was successfully tested and verified on April 8, 2025, with build #6 completing in 24 seconds.

## How We Test the Application

The workflow includes an automated test to verify the application is running correctly:

1. The container is started with port 3000 exposed
2. The workflow waits 5 seconds for the application to initialize
3. An HTTP request is made to the application using curl
4. The response is checked to confirm it contains "Hello from nodejs-demo-app"
5. If the expected text is found, the test passes; otherwise, the workflow fails

This ensures that every Docker image published to Docker Hub contains a properly functioning application.

## Required Secrets

To use this workflow, you must add these secrets to your GitHub repository:

- `DOCKER_HUB_USERNAME`: Your Docker Hub username
- `DOCKER_HUB_TOKEN`: A Docker Hub access token (not your password)

## Docker Hub Repository

The application is published to Docker Hub and available at:
[https://hub.docker.com/repository/docker/slayerop15/nodejs-demo-app/general](https://hub.docker.com/repository/docker/slayerop15/nodejs-demo-app/general)

## Usage

After the workflow completes successfully, you can run the application with:

```bash
docker pull slayerop15/nodejs-demo-app:latest
docker run -p 3000:3000 slayerop15/nodejs-demo-app
```

Then access the application at http://localhost:3000