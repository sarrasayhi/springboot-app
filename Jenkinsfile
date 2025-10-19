pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo '📦 Cloning repository...'
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

                echo '⏳ Waiting for MySQL readiness...'
                sh '''
                    for i in {1..60}; do
                      if docker exec mysql-db mysqladmin ping -uroot -proot --silent; then
                        echo "✅ MySQL is ready!"
                        exit 0
                      fi
                      echo "⌛ Waiting for MySQL... ($i/60)"
                      sleep 3
                    done

                    echo "❌ MySQL failed to start in time"
                    docker logs mysql-db
                    exit 1
                '''
            }
        }

        stage('Build & Test with Maven (in Docker)') {
            steps {
                echo '⚙️ Running Maven build and tests inside Docker (same network)...'
                sh '''
                    docker run --rm \
                      --network ${DOCKER_NETWORK} \
                      -v $PWD:/app -w /app \
                      maven:3.9.9-eclipse-temurin-17 \
                      mvn clean verify \
                        -Dspring.datasource.url=jdbc:mysql://mysql-db:3306/studentdb \
                        -Dspring.datasource.username=root \
                        -Dspring.datasource.password=root
                '''
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    echo '🔍 Running SonarQube analysis...'
                    sh '''
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.projectKey=springboot-app \
                          -Dsonar.sources=src/main/java \
                          -Dsonar.java.binaries=target/classes
                    '''
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
                    echo '🚀 Deploying Spring Boot and MySQL containers...'

                    sh 'docker rm -f mysql-db springboot-container || true'
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    sh '''
                        docker run -d --name mysql-db \
                          --network ${DOCKER_NETWORK} \
                          -e MYSQL_ROOT_PASSWORD=root \
                          -e MYSQL_DATABASE=studentdb \
                          -p 3306:3306 \
                          mysql:8.0
                    '''

                    echo '⏳ Waiting for MySQL...'
                    sh '''
                        for i in {1..60}; do
                          if docker exec mysql-db mysqladmin ping -uroot -proot --silent; then
                            echo "✅ MySQL is ready!"
                            exit 0
                          fi
                          echo "⌛ Waiting... ($i/60)"
                          sleep 3
                        done
                        echo "❌ MySQL failed to start"
                        docker logs mysql-db
                        exit 1
                    '''

                    sh '''
                        docker run -d --name springboot-container \
                          --network ${DOCKER_NETWORK} \
                          -p 9090:8081 \
                          -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/studentdb \
                          -e SPRING_DATASOURCE_USERNAME=root \
                          -e SPRING_DATASOURCE_PASSWORD=root \
                          ${DOCKER_IMAGE}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
            sh 'docker logs mysql-db || true'
        }
    }
}

