pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
        SONARQUBE_ENV = "SonarQube" // The SonarQube server name configured in Jenkins
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo '📥 Cloning repository...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Start MySQL for Tests') {
            steps {
                echo '🧩 Starting MySQL for integration tests...'
                sh '''
                    docker network create ${DOCKER_NETWORK} || true
                    docker rm -f mysql-db || true
                    docker run -d --name mysql-db \
                      --network ${DOCKER_NETWORK} \
                      -e MYSQL_ROOT_PASSWORD=root \
                      -e MYSQL_DATABASE=studentdb \
                      -p 3306:3306 \
                      mysql:8.0
                '''

                echo '⏳ Waiting for MySQL to be ready...'
                sh '''
                    for i in {1..25}; do
                      if docker exec mysql-db mysqladmin ping -h"localhost" -uroot -proot --silent; then
                        echo "✅ MySQL is ready!"
                        exit 0
                      fi
                      echo "⏳ Waiting ($i/25)..."
                      sleep 3
                    done
                    echo "❌ MySQL failed to start in time"
                    exit 1
                '''
            }
        }

        stage('Build and Test with Maven') {
            steps {
                echo '⚙️ Building and testing the Spring Boot app...'
                sh 'mvn clean verify -DskipITs=false'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '🔍 Running SonarQube analysis...'
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Building Docker image...'
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Deploy Containers') {
            steps {
                script {
                    echo '🚀 Deploying containers on Docker network...'

                    // Ensure network exists
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Remove old containers
                    sh 'docker rm -f mysql-db springboot-container || true'

                    // Start MySQL again for production (reuse network)
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8.0
                    '''

                    // Wait for MySQL readiness (again)
                    sh '''
                        echo "⏳ Waiting for MySQL to be production-ready..."
                        for i in {1..25}; do
                          if docker exec mysql-db mysqladmin ping -h"localhost" -uroot -proot --silent; then
                            echo "✅ MySQL is ready for production!"
                            exit 0
                          fi
                          echo "⏳ Waiting ($i/25)..."
                          sleep 3
                        done
                        echo "❌ MySQL failed to start in time"
                        exit 1
                    '''

                    // Start Spring Boot container
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
            echo '✅ Pipeline completed successfully! App is running on http://localhost:9090/student'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
            sh 'docker logs mysql-db || true'
        }
    }
}

