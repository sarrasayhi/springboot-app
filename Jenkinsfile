pipeline {
    agent any

    environment {
        DOCKER_IMAGE   = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📦 Checking out source code...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build JAR (skip tests)') {
            steps {
                echo '⚙️ Building Spring Boot JAR (tests skipped)'
                sh 'mvn clean package -DskipTests=true'
            }
        }

        stage('Docker Compose Up') {
            steps {
                echo '🧱 Starting containers with Docker Compose v2'
                sh '''
                    echo "🧹 Cleaning up any previous containers and networks..."
                    docker compose down || true
                    docker rm -f mysql-db springboot-container || true
                    docker network rm ${DOCKER_NETWORK} || true
                    docker network create ${DOCKER_NETWORK} || true

                    echo "🚀 Bringing up new containers..."
                    docker compose up -d --build
                '''
            }
        }

        stage('Wait for MySQL Readiness') {
            steps {
                echo '⏳ Waiting for MySQL to become ready (port check)...'
                sh '''
                    MAX_RETRIES=40
                    RETRY_DELAY=3
                    echo "🔍 Checking MySQL readiness on port 3306..."
                    for i in $(seq 1 $MAX_RETRIES); do
                        if docker exec mysql-db bash -c "exec 3<>/dev/tcp/127.0.0.1/3306" >/dev/null 2>&1; then
                            echo "✅ MySQL is ready!"
                            exit 0
                        fi
                        echo "⏳ Waiting for MySQL... ($i/$MAX_RETRIES)"
                        sleep $RETRY_DELAY
                    done
                    echo "❌ MySQL did not become ready in time"
                    docker logs mysql-db || true
                    exit 1
                '''
            }
        }

        stage('Verify Spring Boot Startup') {
            steps {
                echo '🔍 Checking if Spring Boot container is healthy...'
                sh '''
                    echo "⏳ Waiting for Spring Boot container startup..."
                    sleep 15
                    if docker logs springboot-container | grep -q "Started"; then
                        echo "✅ Spring Boot started successfully!"
                    else
                        echo "❌ Spring Boot failed to start!"
                        docker logs springboot-container || true
                        exit 1
                    fi
                '''
            }
        }

        stage('Verify Application Endpoint') {
            steps {
                echo '🌐 Verifying Spring Boot application endpoint...'
                sh '''
                    echo "⏳ Waiting 10s before checking endpoint..."
                    sleep 10
                    if curl -s http://localhost:9090/actuator/health | grep -q "UP"; then
                        echo "✅ Application is UP and responding!"
                    else
                        echo "❌ Application is not responding on port 9090!"
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
            sh '''
                docker ps
            '''
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
            sh '''
                echo "📋 MySQL logs:"
                docker logs mysql-db || true
                echo "📋 Spring Boot logs:"
                docker logs springboot-container || true
            '''
        }
        always {
            echo '🧹 Cleaning up containers and networks...'
            sh '''
                docker compose down || true
                docker rm -f mysql-db springboot-container || true
                docker network rm ${DOCKER_NETWORK} || true
            '''
        }
    }
}

