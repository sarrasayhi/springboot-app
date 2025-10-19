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

                echo '⏳ Waiting 20s before checking MySQL readiness...'
                sh 'sleep 20'

                echo '🔍 Checking MySQL readiness...'
                sh '''
                    for i in {1..40}; do
                      if docker exec mysql-db mysql -uroot -proot -e "SELECT 1" >/dev/null 2>&1; then
                        echo "✅ MySQL is ready!"
                        exit 0
                      fi
                      echo "⏳ Waiting for MySQL... ($i/40)"
                      sleep 3
                    done
                    echo "❌ MySQL failed to start in time"
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
                    sh '${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=springboot-app \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target/classes'
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

                    // Wait for DB
                    echo '⏳ Waiting for MySQL to become ready...'
                    sh 'sleep 25'

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

