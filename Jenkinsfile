pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
        SONARQUBE_ENV = "sonarqube"
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
                script {
                    echo '🐬 Starting MySQL container for integration tests...'

                    // Ensure network exists
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Remove any previous container
                    sh 'docker rm -f mysql-db || true'

                    // Start MySQL container
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8
                    '''

                    // 🔄 Add a robust readiness check
                    sh '''
                        echo "⏳ Waiting for MySQL to become ready..."
                        ATTEMPTS=0
                        until docker exec mysql-db mysqladmin ping -h"mysql-db" -uroot -proot --silent; do
                            ATTEMPTS=$((ATTEMPTS+1))
                            if [ $ATTEMPTS -gt 20 ]; then
                                echo "❌ MySQL failed to start after waiting 60s."
                                docker logs mysql-db
                                exit 1
                            fi
                            echo "MySQL not ready yet... retrying in 3s"
                            sleep 3
                        done
                        echo "✅ MySQL is ready and accepting connections!"
                    '''
                }
            }
        }

        stage('Build & Test with Maven') {
            steps {
                echo '🧪 Running Maven tests with real MySQL connection...'
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    echo '🔍 Running SonarQube analysis...'
                    sh 'mvn sonar:sonar -Dsonar.projectKey=springboot-app'
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
                    echo '🚀 Deploying Spring Boot container...'

                    // Remove old app container if it exists
                    sh 'docker rm -f springboot-container || true'

                    // Start Spring Boot app on the same network as MySQL
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
            echo '✅ Pipeline completed successfully! App running at: http://localhost:9090/student'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check Jenkins logs for details.'
            sh 'docker logs mysql-db || true'
        }
    }
}

