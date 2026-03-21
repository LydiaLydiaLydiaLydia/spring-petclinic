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

        stage('Deploy to Test Server') {
            steps {
                echo 'Deploying application...'
                script {
                    // Copy JAR to a known location
                    sh '''
                        mkdir -p /var/jenkins_home/deploy
                        cp target/*.jar /var/jenkins_home/deploy/petclinic.jar
                        echo "JAR copied to deploy directory"
                    '''
                    
                    // Restart the PetClinic container to pick up new JAR
                    sh '''
                        # The docker command below runs from within the jenkins container
                        # but talks to the DinD container which manages all containers
                        docker restart petclinic-app || echo "Container will start on first deploy"
                    '''
                    
                    // Wait for it to be healthy
                    sh '''
                        echo "Waiting for application to start..."
                        COUNTER=0
                        MAX_ATTEMPTS=40
                        
                        while [ $COUNTER -lt $MAX_ATTEMPTS ]; do
                            if curl -s http://petclinic-app:8080 > /dev/null 2>&1; then
                                echo "✓ Application is up!"
                                exit 0
                            fi
                            COUNTER=$((COUNTER + 1))
                            echo "Waiting... attempt $COUNTER/$MAX_ATTEMPTS"
                            sleep 3
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
                }
                failure {
                    echo '✗ Deployment failed'
                }
            }
        }
    }

}