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
        GCP_HOST = '34.78.149.166'
        GCP_USER = 'lydia_sheehan2'
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

        stage('Integration Tests') {
            steps {
                echo 'Running integration tests...'
                sh 'mvn failsafe:integration-test failsafe:verify -DDOCKER_HOST=tcp://docker:2376 -DDOCKER_TLS_VERIFY=1'
            }
            post {
                always {
                    junit '**/target/failsafe-reports/*.xml'
                    echo 'Integration test results published'
                }
                success {
                    echo 'All integration tests passed!'
                }
                failure {
                    echo 'Integration tests failed!'
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
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml,target/site/jacoco-it/jacoco.xml
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

        stage('Configure Environments with Ansible') {
            steps {
                echo 'Configuring deployment environment...'
                sshagent(['gcp-vm-ssh']) {
                    sh 'cd ansible && ansible-playbook playbook.yml'
                }
            }
            post {
                success {
                    echo 'SUCCESS - environments configured with Ansible'
                }
            }
        }


        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application...'

                sh '''
                    # Kill any process already using the port
                    kill $(cat /tmp/${APP_NAME}.pid) 2>/dev/null || true
                    
                    # Run the JAR in the background
                    nohup java -jar target/*.jar \
                        --server.port=${DEPLOY_PORT} \
                        > /tmp/${APP_NAME}.log 2>&1 &
                    
                    echo $! > /tmp/${APP_NAME}.pid
                    
                    # Wait for the application to start
                    echo 'Waiting for application to start...'

                    # Retry health check for up to 60 seconds
                    for i in $(seq 1 12); do
                        sleep 5
                        echo "Health check attempt $i/12..."
                        curl --fail --silent http://localhost:${DEPLOY_PORT}/actuator/health && exit 0
                    done
                    
                    echo 'Health check failed after 60 seconds!'
                    exit 1
                '''
                
            }
            post {
                success {
                    echo 'Application deployed successfully!'
                    echo 'Access at: http://localhost:8090'
                }
                failure {
                    echo 'Deployment failed!'
                }
            }
        }
        
    }

}