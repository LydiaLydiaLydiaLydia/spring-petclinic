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
                sh 'mvn failsafe:integration-test failsafe:verify'
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
                          -Dsonar.host.url=https://sonarcloud.io
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


        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application...'

                sh '''
                    # Kill any process already using the port
                    fuser -k ${DEPLOY_PORT}/tcp || true
                    
                    # Run the JAR in the background
                    nohup java -jar target/*.jar \
                        --server.port=${DEPLOY_PORT} \
                        > /tmp/${APP_NAME}.log 2>&1 &
                    
                    echo $! > /tmp/${APP_NAME}.pid
                    
                    # Wait for the application to start
                    echo 'Waiting for application to start...'
                    sleep 15
                    
                    # Health check
                    curl --fail http://localhost:${DEPLOY_PORT}/actuator/health || \
                        (echo 'Health check failed!' && exit 1)
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