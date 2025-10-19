pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Start MySQL for Tests') {
            steps {
                script {
                    echo 'Starting MySQL container for integration tests...'

                    // Create network if missing
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Remove old MySQL container if exists
                    sh 'docker rm -f mysql-db || true'

                    // Start MySQL
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8
                    '''

                    // Wait for MySQL to initialize
                    sh 'echo "⏳ Waiting for MySQL to start..." && sleep 25'
                }
            }
        }

        stage('Build & Test with Maven') {
            steps {
                echo 'Running Maven tests with real MySQL...'
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    echo 'Running SonarQube analysis...'
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Deploy Containers') {
            steps {
                script {
                    echo 'Deploying Spring Boot app and MySQL on Docker network...'

                    // Remove old containers
                    sh 'docker rm -f mysql-db springboot-container || true'

                    // Start MySQL again (fresh)
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8
                    '''

                    // Wait a bit before starting app
                    sh 'sleep 25'

                    // Start the app container
                    sh '''
                        docker run -d --name springboot-container \
                        --network ${DOCKER_NETWORK} \
                        -p 9090:8081 \
                        ${DOCKER_IMAGE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build, Test, and Deploy succeeded!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check Jenkins logs.'
        }
    }
}

