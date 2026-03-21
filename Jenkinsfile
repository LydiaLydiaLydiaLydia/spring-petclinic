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
                        if [ -f /var/jenkins_home/petclinic.pid ]; then
                            PID=$(cat /var/jenkins_home/petclinic.pid)
                            if ps -p $PID > /dev/null 2>&1; then
                                echo "Stopping existing application (PID: $PID)"
                                kill $PID
                                sleep 3
                            fi
                            rm -f /var/jenkins_home/petclinic.pid
                        fi
                    '''
                    
                    // Start using setsid to fully detach from Jenkins
                    sh '''
                        cd /var/jenkins_home/workspace/PetClinic-Pipeline
                        
                        # Use setsid to completely detach the process
                        setsid java -jar target/*.jar --server.port=8090 \
                            > /var/jenkins_home/petclinic.log 2>&1 < /dev/null &
                        
                        # Save the PID
                        echo $! > /var/jenkins_home/petclinic.pid
                        
                        echo "Started PetClinic with PID: $(cat /var/jenkins_home/petclinic.pid)"
                        
                        # Wait a moment for it to start
                        sleep 5
                    '''
                    
                    // Verify it's still running after detaching
                    sh '''
                        PID=$(cat /var/jenkins_home/petclinic.pid)
                        if ps -p $PID > /dev/null 2>&1; then
                            echo "✓ Process is running with PID: $PID"
                        else
                            echo "✗ Process died after starting"
                            tail -30 /var/jenkins_home/petclinic.log
                            exit 1
                        fi
                    '''
                    
                    // Health check
                    sh '''
                        echo "Waiting for application to respond..."
                        COUNTER=0
                        MAX_ATTEMPTS=30
                        
                        while [ $COUNTER -lt $MAX_ATTEMPTS ]; do
                            if curl -s http://localhost:8090 > /dev/null 2>&1; then
                                echo "✓ Application is up and responding!"
                                
                                # Verify process is still running
                                PID=$(cat /var/jenkins_home/petclinic.pid)
                                if ps -p $PID > /dev/null 2>&1; then
                                    echo "✓ Process still running (PID: $PID)"
                                else
                                    echo "✗ Warning: Process not found but app responded"
                                fi
                                
                                exit 0
                            fi
                            COUNTER=$((COUNTER + 1))
                            echo "Still waiting... attempt $COUNTER/$MAX_ATTEMPTS"
                            sleep 2
                        done
                        
                        echo "✗ Application failed to start"
                        PID=$(cat /var/jenkins_home/petclinic.pid)
                        ps -p $PID || echo "Process not running"
                        echo "Last 30 lines of log:"
                        tail -30 /var/jenkins_home/petclinic.log
                        exit 1
                    '''
                }
            }
            post {
                success {
                    echo '✓ Application deployed and verified running'
                    echo 'Access at: http://localhost:8090'
                }
                failure {
                    echo '✗ Deployment failed'
                }
            }
        }
    }

}