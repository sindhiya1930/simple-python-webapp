// Define environment variables for the pipeline
def dockerImageName = 'simple-python-webapp' // e.g., simple-python-webapp
def dockerHubUsername = 'sindhiya1930'
def eksClusterName = 'skillfyme' // e.g., my-dev-eks-cluster
def awsRegion = 'eu-north-1' // e.g., us-east-1
def k8sNamespace = 'default' // e.g., default

pipeline {
    // Agent is set to 'any' to run on the primary Jenkins node.
    // For production, consider using a dedicated Docker agent with necessary tools.
    agent any
    // Stages of the CI/CD pipeline
    stages {
        // Stage 1: Git Clone
        stage('Git Clone') {
            steps {
                script {
                    echo "Cloning repository..."
                    // Use git plugin to clone the repository
                    git url: 'https://github.com/sindhiya1930/simple-python-webapp.git', branch: 'master'
                    echo "Repository cloned successfully."
                }
            }
        }

        // stage('Run Unit Tests') {
        //     steps {
        //         script {
        //             // Ensure Python and pip are available on the Jenkins agent.
        //             // Install dependencies required for tests.
        //             sh 'python3 -m venv venv'
        //             sh 'source venv/bin/activate'
        //             sh 'pip install -r requirements.txt'
        //             // Run the unit tests using the unittest module.
        //             // The '-v' flag provides verbose output.
        //             sh 'python -m unittest -v test_app.py'
        //         }
        //     }
        // }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    // Get current commit SHA for image tagging
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    // Define the Docker image tag
                    def imageTag = "${dockerHubUsername}/${dockerImageName}:latest"

                    // Build the Docker image. The `docker.build` command requires Docker daemon access.
                    // This leverages the host's Docker daemon via the mounted socket.
                    docker.build("${imageTag}", ".")
                    echo "Docker image built: ${imageTag}"
                    // Store the image tag for later stages
                    env.DOCKER_IMAGE_TAG = imageTag
                }
            }
        }

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
        
        stage('Configure AWS Credentials') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws-credentials', roleAccount: null, roleDuration: 900, roleExternalId: null, roleSessionName: null)]) {
                        sh "echo 'AWS credentials configured.'"
                    }
                }
            }
        }

        stage('Configure EKS') {
            steps {
                script {
                    echo "Deploying to Kubernetes in AWS EKS..."
                    sh "aws eks update-kubeconfig --name ${eksClusterName} --region ${awsRegion}"
                    echo "Kubectl configured for EKS cluster: ${eksClusterName}"
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                // Example: Deploy a Kubernetes manifest to EKS
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }
    }
}
