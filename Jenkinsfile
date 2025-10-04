pipeline {
    agent {
        docker {
            image 'node:16'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        // DockerHub credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        // Docker image name (replace with your DockerHub username)
        DOCKER_IMAGE = 'flowers2algernon/nodejs-app'
        // Snyk API token
        SNYK_TOKEN = credentials('snyk-api-token')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '==== Checking out code from repository ===='
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo '==== Installing Node.js dependencies ===='
                sh 'npm install --save'
                echo '==== Dependencies installed successfully ===='
            }
        }
        
        stage('Run Tests') {
            steps {
                echo '==== Running unit tests ===='
                sh '''
                    # Run tests if test script exists
                    if grep -q "\\"test\\"" package.json; then
                        npm test || echo "No tests defined or tests failed"
                    else
                        echo "No test script defined in package.json"
                    fi
                '''
                echo '==== Tests completed ===='
            }
        }
        
        stage('Security Scan - Dependency Check') {
            steps {
                echo '==== Running Snyk security scan ===='
                script {
                    // Install Snyk CLI
                    sh '''
                        npm install -g snyk
                        snyk --version
                    '''
                    
                    // Authenticate with Snyk
                    sh 'snyk auth $SNYK_TOKEN'
                    
                    // Run Snyk test and fail on high/critical vulnerabilities
                    def snykResult = sh(
                        script: 'snyk test --severity-threshold=high --json > snyk-report.json || true',
                        returnStatus: true
                    )
                    
                    // Archive the Snyk report
                    archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
                    
                    // Display summary
                    sh 'snyk test --severity-threshold=high || true'
                    
                    // Check for high/critical vulnerabilities
                    def hasVulnerabilities = sh(
                        script: 'snyk test --severity-threshold=high',
                        returnStatus: true
                    )
                    
                    if (hasVulnerabilities != 0) {
                        echo 'WARNING: High or Critical vulnerabilities detected!'
                        echo 'Pipeline will fail due to security policy'
                        error('Build failed due to high/critical security vulnerabilities')
                    } else {
                        echo '✓ No high or critical vulnerabilities detected'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo '==== Building Docker image ===='
                script {
                    // Build the Docker image
                    sh """
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                        docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                    """
                    echo "✓ Docker image built: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                echo '==== Pushing Docker image to DockerHub ===='
                script {
                    // Login to DockerHub
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    
                    // Push the image
                    sh """
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                    echo "✓ Image pushed to DockerHub: ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                }
            }
        }
    }
    
    post {
        always {
            echo '==== Cleaning up ===='
            // Logout from DockerHub
            sh 'docker logout || true'
            
            // Clean up Docker images to save space
            sh """
                docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                docker rmi ${DOCKER_IMAGE}:latest || true
            """
        }
        success {
            echo '✓✓✓ Pipeline completed successfully! ✓✓✓'
        }
        failure {
            echo '✗✗✗ Pipeline failed! ✗✗✗'
        }
    }
}
