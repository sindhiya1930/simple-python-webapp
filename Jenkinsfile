// Define environment variables for the pipeline
def dockerImageName = 'simple-python-webapp' // e.g., simple-python-webapp
def dockerHubUsername = 'sindhiya1930'
def eksClusterName = 'skillfyme' // e.g., my-dev-eks-cluster
def awsRegion = 'eu-west-1' // e.g., us-east-1
def k8sNamespace = 'default' // e.g., default

pipeline {
    // Agent is set to 'any' to run on the primary Jenkins node.
    // For production, consider using a dedicated Docker agent with necessary tools.
    agent any

    // Options for the pipeline
    options {
        // BuildDiscarder to keep a limited number of builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timeout for the entire pipeline
        timeout(time: 30, unit: 'MINUTES')
        // Enable timestamps in console output
        timestamps()
    }

    // Define environment variables specific to credentials
    environment {
        // AWS credentials (using the ID configured in Jenkins Credentials)
        AWS_ACCESS_KEY_ID = credentials('aws-credentials').getAWSAccessKeyId()
        AWS_SECRET_ACCESS_KEY = credentials('aws-credentials').getSecretAccessKey()
        // DOCKER_USERNAME and DOCKER_PASSWORD for Docker Hub login (using the ID configured)
        DOCKER_USERNAME = credentials('docker-hub-credentials').getUsername()
        DOCKER_PASSWORD = credentials('docker-hub-credentials').getPassword()
    }

    // Stages of the CI/CD pipeline
    stages {
        // Stage 1: Git Clone
        stage('Git Clone') {
            steps {
                script {
                    echo "Cloning repository..."
                    // Use git plugin to clone the repository
                    git url: 'YOUR_GITHUB_REPO_URL', branch: 'main'
                    echo "Repository cloned successfully."
                }
            }
        }

        // Stage 2: Install Python Dependencies (Build Artifact)
        stage('Install Dependencies') {
            steps {
                script {
                    echo "Installing Python dependencies..."
                    // Use a Docker container to run Python commands
                    // This ensures a clean and consistent environment for dependencies
                    docker.image('python:3.9-slim-buster').inside {
                        sh 'pip install --no-cache-dir -r requirements.txt'
                    }
                    echo "Python dependencies installed."
                }
            }
        }

        // Stage 3: Run Unit Tests
        stage('Run Unit Tests') {
            steps {
                script {
                    echo "Running unit tests..."
                    // Run tests inside the same Python Docker container
                    docker.image('python:3.9-slim-buster').inside {
                        sh 'python -m unittest test_app.py'
                    }
                    echo "Unit tests completed."
                }
            }
        }

        // Stage 4: Build Docker Image
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    // Get current commit SHA for image tagging
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    // Define the Docker image tag
                    def imageTag = "${dockerHubUsername}/${dockerImageName}:${gitCommit}"

                    // Build the Docker image. The `docker.build` command requires Docker daemon access.
                    // This leverages the host's Docker daemon via the mounted socket.
                    docker.build("${imageTag}", ".")
                    echo "Docker image built: ${imageTag}"
                    // Store the image tag for later stages
                    env.DOCKER_IMAGE_TAG = imageTag
                }
            }
        }

        // Stage 5: Push Docker Image to Docker Hub
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "Pushing Docker image to Docker Hub..."
                    // Use withRegistry to log in to Docker Hub using the configured credentials
                    docker.withRegistry("https://index.docker.io/v1/", 'docker-hub-credentials') {
                        def image = docker.image(env.DOCKER_IMAGE_TAG)
                        // Push the image
                        image.push()
                    }
                    echo "Docker image pushed to Docker Hub: ${env.DOCKER_IMAGE_TAG}"
                }
            }
        }

        // Stage 6: Deploy to Kubernetes in AWS EKS
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying to Kubernetes in AWS EKS..."

                    // 1. Configure kubectl for EKS
                    // This command uses AWS CLI to update the kubeconfig file on the Jenkins host.
                    // It assumes AWS CLI and `aws-iam-authenticator` (or a recent AWS CLI version that includes it)
                    // are installed and configured on the host where Jenkins Docker container is running.
                    // Ensure the IAM user associated with AWS_ACCESS_KEY_ID has permissions to describe and update EKS clusters.
                    sh "aws eks update-kubeconfig --name ${eksClusterName} --region ${awsRegion}"
                    echo "Kubectl configured for EKS cluster: ${eksClusterName}"

                    // 2. Create Kubernetes Deployment and Service YAML files dynamically
                    // This allows us to inject the correct Docker image tag
                    // Create deployment.yaml
                    def deploymentYaml = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${dockerImageName}-deployment
  namespace: ${k8sNamespace}
  labels:
    app: ${dockerImageName}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${dockerImageName}
  template:
    metadata:
      labels:
        app: ${dockerImageName}
    spec:
      containers:
      - name: ${dockerImageName}
        image: ${env.DOCKER_IMAGE_TAG}
        ports:
        - containerPort: 5000
"""
                    writeFile file: 'deployment.yaml', text: deploymentYaml
                    echo "deployment.yaml created."

                    // Create service.yaml
                    def serviceYaml = """
apiVersion: v1
kind: Service
metadata:
  name: ${dockerImageName}-service
  namespace: ${k8sNamespace}
spec:
  selector:
    app: ${dockerImageName}
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
  type: LoadBalancer # Use LoadBalancer for external access, NodePort or ClusterIP for internal
"""
                    writeFile file: 'service.yaml', text: serviceYaml
                    echo "service.yaml created."

                    // 3. Apply Kubernetes configurations
                    // Apply deployment first
                    sh "kubectl apply -f deployment.yaml -n ${k8sNamespace}"
                    echo "Deployment applied for ${dockerImageName}."

                    // Apply service
                    sh "kubectl apply -f service.yaml -n ${k8sNamespace}"
                    echo "Service applied for ${dockerImageName}."

                    echo "Deployment to Kubernetes completed."
                }
            }
        }
    }

    // Post-build actions
    post {
        always {
            echo "Pipeline finished."
            // Clean up workspace after build
            deleteDir()
        }
        success {
            echo "Pipeline succeeded!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
