pipeline {
    agent any
    environment {
        DOCKER_IMAGE    = "ahmeddevop/petclinic"
        DOCKER_TAG      = "${env.BUILD_NUMBER}"
        NEXUS_HOSTED    = "localhost:30082"
        NEXUS_GROUP     = "localhost:30083"
        NEXUS_IMAGE     = "localhost:30082/petclinic"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Test') {
            steps {
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
                sh 'mvn checkstyle:check || true'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
            }
        }
        stage('Push to Docker Hub') {
            steps {
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
        stage('Push to Nexus Hosted') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo \$NEXUS_PASS | docker login ${NEXUS_HOSTED} \
                            -u \$NEXUS_USER --password-stdin
                        docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${NEXUS_IMAGE}:${DOCKER_TAG}
                        docker tag ${DOCKER_IMAGE}:latest ${NEXUS_IMAGE}:latest
                        docker push ${NEXUS_IMAGE}:${DOCKER_TAG}
                        docker push ${NEXUS_IMAGE}:latest
                    """
                }
            }
        }
        stage('Deploy') {
            steps {
                sh """
                    docker stop petclinic || true
                    docker rm   petclinic || true
                    docker run -d \
                        --name petclinic \
                        -p 8090:8080 \
                        --restart unless-stopped \
                        ${NEXUS_GROUP}/petclinic:latest
                """
            }
        }
        stage('Verify') {
            steps {
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
            echo '❌ Pipeline failed!'
        }
        always {
            sh "docker logout ${NEXUS_HOSTED} || true"
            sh 'docker logout || true'
        }
    }
}
