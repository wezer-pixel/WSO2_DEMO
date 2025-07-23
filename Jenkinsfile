pipeline {
    agent any

    environment {
        // IMPORTANT: Change this to your Docker Hub username
        DOCKERHUB_USERNAME       = 'your-dockerhub-username'
        // The ID of your Docker Hub credentials stored in Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
    }

    stages {
        stage('Build and Push Backend Images') {
            failFast true // if one build fails, stop the others
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
        def serviceName = servicePath.split('/').last()
        // Tag the image with the build number for versioning
        def imageName = "${env.DOCKERHUB_USERNAME}/${serviceName}:${env.BUILD_NUMBER}"
        
        echo "--- Building and Pushing ${imageName} from path ${servicePath} ---"

        dir(servicePath) {
            sh 'chmod +x mvnw'
            sh './mvnw clean package -DskipTests'
            def dockerImage = docker.build(imageName, '.')
            docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDENTIALS_ID) {
                dockerImage.push()
            }
            echo "--- Successfully pushed ${imageName} ---"
        }
    }
}