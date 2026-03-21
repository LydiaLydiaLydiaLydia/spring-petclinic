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
        
        // NEW STAGE - Deploy to Test Server
        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application to test environment...'
                script {
                    // Stop any existing instance
                    sh '''
                        PID=$(ps aux | grep 'spring-petclinic' | grep -v grep | awk '{print $2}' | head -1)
                        if [ ! -z "$PID" ]; then
                            echo "Stopping existing application (PID: $PID)"
                            kill -9 $PID || true
                            sleep 2
                        else
                            echo "No existing application running"
                        fi
                    '''
                    
                    // Start the application in background
                    sh """
                        echo "Starting application on port ${DEPLOY_PORT}..."
                        nohup java -jar target/*.jar \
                          --server.port=${DEPLOY_PORT} \
                          > /var/jenkins_home/petclinic.log 2>&1 &
                        
                        echo \$! > /var/jenkins_home/petclinic.pid
                        echo "Application started with PID: \$(cat /var/jenkins_home/petclinic.pid)"
                    """
                    
                    // Wait for application to start and verify health
                    sh """
                        echo "Waiting for application to start..."
                        for i in {1..30}; do
                            if curl -s http://localhost:${DEPLOY_PORT} > /dev/null 2>&1; then
                                echo "✅ Application is up and responding!"
                                curl -s http://localhost:${DEPLOY_PORT} | grep -q "PetClinic" && echo "✅ PetClinic homepage verified"
                                exit 0
                            fi
                            echo "Still starting... attempt \$i/30"
                            sleep 2
                        done
                        echo "❌ Application failed to start within 60 seconds"
                        cat /var/jenkins_home/petclinic.log
                        exit 1
                    """
                }
            }
            post {
                success {
                    echo '✅ Application deployed successfully!'
                    echo "🌐 Access the application at: http://localhost:${DEPLOY_PORT}"
                }
                failure {
                    echo '❌ Deployment failed - checking logs...'
                    sh 'tail -50 /var/jenkins_home/petclinic.log || echo "No log file found"'
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
        }
        
        success {
            echo 'Build succeeded!'
            emailext (
                subject: "SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    Build Successful!
                    
                    Job: ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Build URL: ${env.BUILD_URL}
                    
                    All stages completed successfully.
                """,
                to: 'your-email@example.com',
                mimeType: 'text/plain'
            )
        }
        
        failure {
            echo 'Build failed!'
            emailext (
                subject: "FAILURE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    Build Failed!
                    
                    Job: ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Build URL: ${env.BUILD_URL}
                    
                    Check console output for details.
                """,
                to: 'your-email@example.com',
                mimeType: 'text/plain'
            )
        }
    }