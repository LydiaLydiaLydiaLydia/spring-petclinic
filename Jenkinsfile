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
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
        }
        success {
            echo 'Build succeeded!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}