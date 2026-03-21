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
        
        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application to test environment...'
                script {
                    // Stop any existing instance
                    sh '''
                        PID=$(ps aux | grep 'spring-petclinic' | grep -v grep | awk '{print $2}')
                        if [ ! -z "$PID" ]; then
                            echo "Stopping existing application (PID: $PID)"
                            kill -9 $PID
                            sleep 2
                        fi
                    '''
                    
                    // Start the application in background
                    sh """
                        nohup java -jar target/*.jar \
                        --server.port=8090 \
                        > /var/jenkins_home/petclinic.log 2>&1 &
                        
                        echo \$! > /var/jenkins_home/petclinic.pid
                        echo "Application started with PID: \$(cat /var/jenkins_home/petclinic.pid)"
                    """
                    
                    // Wait for application to start (FIXED LOOP)
                    sh '''
                        echo "Waiting for application to start..."
                        COUNTER=0
                        MAX_ATTEMPTS=30
                        
                        while [ $COUNTER -lt $MAX_ATTEMPTS ]; do
                            if curl -s http://localhost:8090 > /dev/null 2>&1; then
                                echo "Application is up and responding!"
                                exit 0
                            fi
                            COUNTER=$((COUNTER + 1))
                            echo "Still starting... attempt $COUNTER/$MAX_ATTEMPTS"
                            sleep 2
                        done
                        
                        echo "Application failed to start within 60 seconds"
                        echo "Last 20 lines of log:"
                        tail -20 /var/jenkins_home/petclinic.log
                        exit 1
                    '''
                }
            }
            post {
                success {
                    echo 'Application deployed successfully!'
                    echo 'Access the application at: http://localhost:8090'
                }
                failure {
                    echo 'Deployment failed - check logs above'
                }
            }
        }
    }

}