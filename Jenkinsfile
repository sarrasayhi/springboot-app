pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
        SONARQUBE_ENV  = "SonarQube"
        MYSQL_CONTAINER = "mysql-db"
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
                    docker rm -f ${MYSQL_CONTAINER} || true

                    docker run -d --name ${MYSQL_CONTAINER} \
                      --network ${DOCKER_NETWORK} \
                      -e MYSQL_ROOT_PASSWORD=root \
                      -e MYSQL_DATABASE=studentdb \
                      -p 3306:3306 \
                      mysql:8.0

                    echo "⏳ Waiting for MySQL service to initialize..."
                    for i in {1..60}; do
                      if docker exec ${MYSQL_CONTAINER} bash -c "mysqladmin ping -h localhost -uroot -proot --silent"; then
                        echo "✅ MySQL is ready!"
                        exit 0
                      fi
                      echo "⏳ Waiting ($i/60)..."
                      sleep 3
                    done

                    echo "❌ MySQL did not become ready in time"
                    docker logs ${MYSQL_CONTAINER}
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

                    echo '⏳ Waiting for MySQL to become ready...'
                    for i in {1..60}; do
                      if docker exec mysql-db bash -c "mysqladmin ping -h localhost -uroot -proot --silent"; then
                        echo "✅ MySQL is ready for app startup!"
                        break
                      fi
                      sleep 3
                    done

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
            echo '❌ Pipeline failed! Showing MySQL logs...'
            sh 'docker logs ${MYSQL_CONTAINER} || true'
        }
    }
}

