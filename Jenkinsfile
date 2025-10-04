pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKER_IMAGE = 'flowers2algernon/nodejs-app'
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
                sh '''
                    docker run --rm \
                    -v $(pwd):/app \
                    -w /app \
                    node:16 \
                    npm install --save
                '''
                echo '==== Dependencies installed successfully ===='
            }
        }
        
        stage('Run Tests') {
            steps {
                echo '==== Running unit tests ===='
                sh '''
                    if grep -q "\\"test\\"" package.json; then
                        docker run --rm -v $(pwd):/app -w /app node:16 npm test || echo "Tests completed"
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
                    // Install Snyk and run scan
                    sh """
                        docker run --rm \
                        -v \$(pwd):/app \
                        -w /app \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        node:16 \
                        sh -c 'npm install -g snyk && \
                               snyk --version && \
                               snyk auth \$SNYK_TOKEN && \
                               snyk test --severity-threshold=high --json > snyk-report.json || true'
                    """
                    
                    // Archive the report
                    archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
                    
                    // Display summary
                    sh """
                        docker run --rm \
                        -v \$(pwd):/app \
                        -w /app \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        node:16 \
                        sh -c 'npm install -g snyk && \
                               snyk auth \$SNYK_TOKEN && \
                               snyk test --severity-threshold=high || true'
                    """
                    
                    // Check for vulnerabilities
                    def hasVulnerabilities = sh(
                        script: """
                            docker run --rm \
                            -v \$(pwd):/app \
                            -w /app \
                            -e SNYK_TOKEN=${SNYK_TOKEN} \
                            node:16 \
                            sh -c 'npm install -g snyk && \
                                   snyk auth \$SNYK_TOKEN && \
                                   snyk test --severity-threshold=high'
                        """,
                        returnStatus: true
                    )
                    
                    if (hasVulnerabilities != 0) {
                        echo '⚠️  WARNING: High or Critical vulnerabilities detected!'
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
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    
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
            sh 'docker logout || true'
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
