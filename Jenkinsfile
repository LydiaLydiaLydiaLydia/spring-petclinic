pipeline {
    agent any
    
    tools {
        maven 'Maven 3.9.9'
        jdk 'OpenJDK 21.0.9'
    }
    
    environment {
        APP_NAME = 'spring-petclinic'
        SONAR_PROJECT_KEY = 'LydiaLydiaLydiaLydia_spring-petclinic'
        SONAR_ORGANIZATION = 'lydialydialydialydia'
        RECIPIENT_EMAIL = 'lydia.sheehan2@gmail.com'
        DEPLOY_PORT = '8090'  // Added for deployment
        DOCKER_HUB_USERNAME = 'lydialydialydialydia'
        DOCKER_IMAGE_NAME = "${DOCKER_HUB_USERNAME}/petclinic"
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Compiling the application...'
                sh 'mvn clean compile'
            }
        }
        
        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    echo 'Test results published'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube code analysis...'
                withSonarQubeEnv('SonarCloud') {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.organization=${SONAR_ORGANIZATION} \
                          -Dsonar.host.url=https://sonarcloud.io
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube quality gate result...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging application...'
                sh 'mvn package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                    echo 'Artifacts archived successfully'
                }
            }
        }
        
        stage('Configure Environment with Ansible') {
            steps {
                echo 'Configuring deployment environment...'
                script {
                    sh '''
                        cd ansible
                        ansible-playbook playbook.yml -i inventory.ini
                    '''
                }
            }
            post {
                success {
                    echo '✓ Environment configured successfully'
                }
            }
        }


        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    // Build image with build number tag
                    sh """
                        docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} .
                        docker build -t ${DOCKER_IMAGE_NAME}:latest .
                    """
                }
            }
            post {
                success {
                    echo "Docker image built: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing Docker image to Docker Hub...'
                script {
                    // Login and push
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                            docker push ${DOCKER_IMAGE_NAME}:latest
                            docker logout
                        """
                    }
                }
            }
            post {
                success {
                    echo "Image pushed to Docker Hub: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application from Docker image...'
                script {
                    sh """
                        # Stop and remove old container if exists
                        docker stop petclinic-app || true
                        docker rm petclinic-app || true
                        
                        # Pull the latest image
                        docker pull ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                        
                        # Run the new container
                        docker run -d \
                            --name petclinic-app \
                            -p 8090:8080 \
                            --restart unless-stopped \
                            ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                        
                        echo "Container started with image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    """
                    
                    // Health check - access the container directly
                    sh '''
                        echo "Waiting for application to start..."
                        COUNTER=0
                        MAX_ATTEMPTS=30
                        
                        while [ $COUNTER -lt $MAX_ATTEMPTS ]; do
                            # Check container health directly by accessing port 8080 inside the container
                            if docker exec petclinic-app wget --quiet --tries=1 --spider http://localhost:8080 2>/dev/null; then
                                echo "✓ Application is up and responding!"
                                exit 0
                            fi
                            COUNTER=$((COUNTER + 1))
                            echo "Waiting... attempt $COUNTER/$MAX_ATTEMPTS"
                            sleep 2
                        done
                        
                        echo "✗ Application failed to start"
                        docker logs petclinic-app --tail 50
                        exit 1
                    '''
                }
            }
            post {
                success {
                    echo '✓ Application deployed successfully!'
                    echo 'Access at: http://localhost:8090'
                    echo "Image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
                failure {
                    echo '✗ Deployment failed!'
                }
            }
        }
    }

}