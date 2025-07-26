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
                    which java || echo "❌ Java (JDK) not found - required for Maven builds"
                    which mvn || echo "⚠️ Maven (mvn) not found - will rely on ./mvnw wrapper"
                    if ! command -v yq &> /dev/null
                    then
                        echo "yq not found, installing..."
                        # Add command to install yq based on your agent's OS
                        # For example, on Alpine:
                        apk add --no-cache yq
                    fi
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
                        'dockerfiles/keycloak-setup-script',
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

        stage('Build and Push All Images') {
            parallel {
                // WSO2 Components & Scripts from docker-compose.yml
                stage('mi-runtime') {
                    steps {
                        buildAndPush(
                            servicePath: 'dockerfiles/micro-integrator', 
                            serviceNameOverride: 'mi-runtime', 
                            buildArgs: '--build-arg BASE_IMAGE=wso2/wso2mi:4.2.0'
                        )
                    }
                }
                stage('si-runtime') {
                    steps {
                        buildAndPush(
                            servicePath: 'dockerfiles/streaming-integrator', 
                            serviceNameOverride: 'si-runtime', 
                            buildArgs: '--build-arg BASE_IMAGE=wso2/wso2si:4.0.0-ubuntu'
                        )
                    }
                }
                stage('apim-scripts') {
                    steps {
                        buildAndPush(
                            servicePath: 'dockerfiles/scripts', 
                            serviceNameOverride: 'apim-scripts'
                        )
                    }
                }
                // Backend Services
                stage('train-schedule-backend') {
                    steps {
                        buildAndPush(servicePath: 'backends/train-schedule', serviceNameOverride: 'train-schedule-backend')
                    }
                }
                stage('train-location-simulator') {
                    steps {
                        buildAndPush(servicePath: 'backends/train-location-simulator')
                    }
                }
                stage('telecom-backend') {
                    steps {
                        buildAndPush(servicePath: 'backends/telecom-backends', serviceNameOverride: 'telecom-backend')
                    }
                }
                stage('soap-service-backend') {
                    steps {
                        buildAndPush(servicePath: 'backends/soap-service', serviceNameOverride: 'soap-service-backend')
                    }
                }
                stage('telecom-soap-backend') {
                    steps {
                        buildAndPush(servicePath: 'backends/telecom-soap-service', serviceNameOverride: 'telecom-soap-backend')
                    }
                }
            }
        }
    }
}
/**
 * Helper function to build an app, then build and push a Docker image.
 * @param servicePath The path to the service directory.
 * @param serviceNameOverride Optional name for the image, otherwise derived from path.
 * @param buildArgs Optional arguments for the 'docker build' command.
 */
void buildAndPush(Map params) {
    def servicePath = params.servicePath
    def serviceNameOverride = params.get('serviceNameOverride', '')
    def buildArgs = params.get('buildArgs', '')
    script {
        echo "=== BUILDANDPUSH: Starting for ${servicePath} ==="
        
        def serviceName = serviceNameOverride ?: servicePath.tokenize('/').last()
        // Use the short git commit hash for the image tag
        def imageTag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        def imageName = "${env.DOCKERHUB_USERNAME}/${serviceName}"
        def fullImageNameWithTag = "${imageName}:${imageTag}"
        
        echo "--- Building and Pushing ${fullImageNameWithTag} from path ${servicePath} ---"

        dir(servicePath) {
            // Step 1: Build the application if it's a Maven project
            if (fileExists('pom.xml')) {
                echo "✅ Found pom.xml, running Maven build..."
                // Use mvnw if available, otherwise assume mvn is in PATH
                if (isUnix()) {
                    sh 'chmod +x ./mvnw'
                    sh './mvnw clean package -DskipTests'
                } else {
                    bat 'mvnw.cmd clean package -DskipTests'
                }
                echo "--- Maven build complete for ${serviceName} ---"
            }

            // Step 2: Build and push the Docker image
            if (fileExists('Dockerfile')) {
                echo "✅ Found Dockerfile, building and pushing Docker image..."

                // If a base image is specified in build-args, pull it first for reliability.
                if (buildArgs.contains('BASE_IMAGE')) {
                    def baseImage = buildArgs.split('=')[1].trim()
                    echo "--- Pulling base image ${baseImage} ---"
                    try {
                        sh "docker pull ${baseImage}"
                    } catch (e) {
                        error "Failed to pull base image ${baseImage}. Please check if the image exists and is accessible. Error: ${e.message}"
                    }
                }

                // Build the Docker image
                def dockerImage = docker.build(fullImageNameWithTag, "${buildArgs} .")
                
                // Push the built image
                docker.withRegistry('', env.DOCKERHUB_CREDENTIALS_ID) {
                    dockerImage.push() // Pushes the specific tag (e.g., a1b2c3d)
                    dockerImage.push('latest') // Also pushes the 'latest' tag
                }
                echo "--- Successfully pushed ${fullImageNameWithTag} and ${imageName}:latest ---"
            } else {
                echo "❌ No Dockerfile found in ${servicePath}, skipping Docker build"
            }
        }
    }
}