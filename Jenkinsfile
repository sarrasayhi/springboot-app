pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
        SONARQUBE_ENV  = "SonarQube"
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

                echo '🔍 Waiting for MySQL to be ready (port check)...'
                sh '''
                    for i in {1..60}; do
                      if docker exec mysql-db sh -c "nc -z localhost 3306"; then
                        echo "✅ MySQL is ready on port 3306!"
                        exit 0
                      fi
                      echo "⏳ Waiting for MySQL to open port 3306... ($i/60)"
                      sleep 3
                    done
                    echo "❌ MySQL did not start in time."
                    docker logs mysql-db
                    exit 1
                '''
            }
        }

        stage('Build & Test with Maven') {
            steps {
                echo '⚙️ Building and testing with Maven...'
                sh 'mvn clean verify -DskipTests=false'
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

                    // Stop any running containers
                    sh 'docker rm -f mysql-db springboot-container || true'

                    // Recreate network if missing
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Start MySQL
                    sh '''
                        docker run -d --name mysql-db \
                          --network ${DOCKER_NETWORK} \
                          -e MYSQL_ROOT_PASSWORD=root \
                          -e MYSQL_DATABASE=studentdb \
                          -p 3306:3306 \
                          mysql:8.0
                    '''

                    // Wait for MySQL readiness
                    echo '🔍 Checking MySQL readiness for deployment...'
                    sh '''
                        for i in {1..60}; do
                          if docker exec mysql-db sh -c "nc -z localhost 3306"; then
                            echo "✅ MySQL is ready for app connection!"
                            exit 0
                          fi
                          echo "⏳ Waiting for MySQL... ($i/60)"
                          sleep 3
                        done
                        echo "❌ MySQL did not become ready in time."
                        docker logs mysql-db
                        exit 1
                    '''

                    // Start Spring Boot app
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
            echo '✅ Pipeline completed successfully!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
            sh 'docker logs mysql-db || true'
        }
    }
}

