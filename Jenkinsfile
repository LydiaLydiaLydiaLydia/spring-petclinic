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
            emailext (
                subject: "✅ SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: green;">Build Successful!</h2>
                    
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                    <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><strong>Duration:</strong> ${currentBuild.durationString}</p>
                    
                    <h3>What happened:</h3>
                    <ul>
                        <li>✅ Code compiled successfully</li>
                        <li>✅ All tests passed</li>
                        <li>✅ Code quality gate passed</li>
                        <li>✅ Application packaged</li>
                    </ul>
                    
                    <p><a href="${env.BUILD_URL}console">View Console Output</a></p>
                """,
                to: "${RECIPIENT_EMAIL}",
                mimeType: 'text/html'
            )
        }
        failure {
            echo 'Build failed!'
            emailext (
                subject: "❌ FAILURE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: red;">Build Failed!</h2>
                    
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                    <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    
                    <h3>Action Required:</h3>
                    <p>The build has failed. Please check the console output for details:</p>
                    <p><a href="${env.BUILD_URL}console">View Console Output</a></p>
                    
                    <h3>Recent Changes:</h3>
                    <p>${currentBuild.changeSets.collect { it.items.collect { item -> "- ${item.msg} by ${item.author}" }.join('<br>') }.join('<br>')}</p>
                """,
                to: "${RECIPIENT_EMAIL}",
                mimeType: 'text/html'
            )
        }
        unstable {
            echo '⚠️ Build is unstable'
            emailext (
                subject: "⚠️ UNSTABLE: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: orange;">Build Unstable</h2>
                    
                    <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                    <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                    
                    <p>The build completed but some tests failed or quality gate warnings exist.</p>
                    <p><a href="${env.BUILD_URL}console">View Console Output</a></p>
                """,
                to: "${RECIPIENT_EMAIL}",
                mimeType: 'text/html'
            )
        }
    }
}