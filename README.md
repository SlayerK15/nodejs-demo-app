# Nodejs-Demo-App

A simple Node.js application containerized with Docker and automated with GitHub Actions.

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

## GitHub Actions Workflow

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
