pipeline {
    agent any

    environment {
        // IMPORTANT: Change this to your Docker Hub username
        DOCKERHUB_USERNAME       = 'wezer'
        // The ID of your Docker Hub credentials stored in Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'wso2-docker-token'

        GITOPS_REPO_URL = "https://github.com/wezer-pixel/WSO2_DEMO.git"
        GITOPS_REPO_CREDS_ID = 'wso2-gh-token'
    }

    stages {
        stage('System Check') {
            steps {
                sh '''
                    echo "=== System Information ==="
                    whoami
                    pwd
                    
                    echo "=== Available Commands ==="
                    which docker || echo "❌ Docker not found - needs to be installed"
                    which git || echo "❌ Git not found"
                    which yq || echo "❌ yq not found - will use sed"
                    which sed || echo "✅ sed found"
                    
                    echo "=== Docker Check ==="
                    if command -v docker > /dev/null 2>&1; then
                        docker --version
                        docker ps || echo "Docker daemon not running or permission denied"
                    else
                        echo "Docker is not installed on this system"
                    fi
                '''
            }
        }

        stage('Checkout SCM') {
            steps {
                // This cleans the workspace before checking out the application code
                cleanWs()
                checkout scm
                
                // Add debug output to see directory structure
                sh '''
                    echo "=== Workspace Contents ==="
                    ls -la
                    echo "=== Checking for backends directory ==="
                    ls -la backends/ || echo "No backends directory found"
                    echo "=== Current working directory ==="
                    pwd
                '''
            }
        }

        stage('Debug Stage') {
            steps {
                script {
                    echo "=== DEBUG: This stage should execute ==="
                    echo "DOCKERHUB_USERNAME: ${env.DOCKERHUB_USERNAME}"
                    echo "DOCKERHUB_CREDENTIALS_ID: ${env.DOCKERHUB_CREDENTIALS_ID}"
                    
                    // Check if directories exist
                    def servicePaths = [
                        'dockerfiles/apim',
                        'dockerfiles/backends',
                        'dockerfiles/backends/rest',
                        'dockerfiles/backends/soap',
                        'dockerfiles/backends/keycloack-setup-script',
                        'dockerfiles/scripts',
                        'dockerfiles/streaming-integrator',
                        'dockerfiles/streaming-integrator/build-image'
                    ]
                    
                    servicePaths.each { servicePath ->
                        if (fileExists(servicePath)) {
                            echo "✅ Found directory: ${servicePath}"
                        } else {
                            echo "❌ Missing directory: ${servicePath}"
                        }
                    }
                }
            }
        }

        stage('Build and Push Backend Images') {
            parallel {
                stage('train-schedule') {
                    steps {
                        buildAndPush('backends/train-schedule')
                    }
                }
                stage('train-location-simulator') {
                    steps {
                        buildAndPush('backends/train-location-simulator')
                    }
                }
                stage('telecom-backends') {
                    steps {
                        buildAndPush('backends/telecom-backends')
                    }
                }
                stage('soap-service') {
                    steps {
                        buildAndPush('backends/soap-service')
                    }
                }
                stage('telecom-soap-service') {
                    steps {
                        buildAndPush('backends/telecom-soap-service')
                    }
                }
            }
        }
    }
}

/**
 * Helper function to build a Java app, then build and push a Docker image.
 * @param servicePath The path to the backend service directory.
 */
void buildAndPush(String servicePath) {
    script {
        echo "=== BUILDANDPUSH: Starting for ${servicePath} ==="
        
        // Check if Docker is available
        def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0

        if (!dockerAvailable) {
            error "Docker is not installed on this Jenkins agent. Please install Docker first."
        }

        def serviceName = servicePath.split('/').last()
        // Use the short git commit hash for the image tag
        def imageTag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()

        // Correct image name format: <username>/<service-name>:<tag>
        def imageName = "${env.DOCKERHUB_USERNAME}/${serviceName}:${imageTag}"
        
        echo "--- Building and Pushing ${imageName} from path ${servicePath} ---"

        dir(servicePath) {
            // List directory contents for debugging
            sh 'echo "=== Directory contents ==="; ls -la'
                        
            // Check if Dockerfile exists
            if (fileExists('Dockerfile')) {
                echo "✅ Found Dockerfile, building and pushing Docker image..."
                // Build the Docker image
                def dockerImage = docker.build(imageName, '.')
                
                // Push the built image
                docker.withRegistry('https://docker.io', env.DOCKERHUB_CREDENTIALS_ID) {
                    dockerImage.push()  // Push with commit hash tag
                    dockerImage.push('latest')  // Also push as latest tag
                }
                echo "--- Successfully pushed ${imageName} ---"
            } else {
                echo "❌ No Dockerfile found in ${servicePath}, skipping Docker build"
            }
        }
    }
}