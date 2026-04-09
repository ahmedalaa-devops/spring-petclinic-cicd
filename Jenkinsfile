pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "ahmeddevop/petclinic"
        DOCKER_TAG   = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Cloning repository...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Building with Maven...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Running unit tests...'
                sh 'mvn test -Dtest="!MySqlIntegrationTests"'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Code Quality Check') {
            steps {
                echo '🔍 Checking code quality...'
                sh 'mvn checkstyle:check || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Building Docker image...'
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo '📤 Pushing to Docker Hub...'
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Deploying container...'
                sh """
                    docker stop petclinic || true
                    docker rm   petclinic || true
                    docker run -d \
                        --name petclinic \
                        -p 8090:8080 \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Verify') {
            steps {
                echo '✔️ Verifying deployment...'
                sh 'sleep 10'
                sh 'curl -f http://localhost:8090 || exit 1'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed — check logs!'
        }
        always {
            sh 'docker logout || true'
        }
    }
}
