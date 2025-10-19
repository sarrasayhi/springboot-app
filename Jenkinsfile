pipeline {
    agent any

    environment {
        APP_NAME = "springboot-app"
        APP_PORT = "8081"
        MYSQL_ROOT_PASSWORD = "root"
        MYSQL_DB = "studentdb"
        MYSQL_USER = "root"
        MYSQL_PASSWORD = "root"
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo "📦 Checking out code from GitHub..."
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build Maven Project') {
            steps {
                echo "🧱 Building Spring Boot project with Maven..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image for Spring Boot..."
                sh '''
                docker build -t ${APP_NAME}:latest .
                '''
            }
        }

        stage('Run MySQL Container') {
            steps {
                echo "🐬 Starting MySQL container..."
                sh '''
                docker rm -f mysql-db || true
                docker run -d --name mysql-db \
                    -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
                    -e MYSQL_DATABASE=${MYSQL_DB} \
                    -p 3306:3306 \
                    mysql:8
                '''
            }
        }

        stage('Wait for MySQL Readiness') {
            steps {
                echo "⏳ Waiting for MySQL to become ready..."
                sh '''
                for i in {1..20}; do
                    if docker exec mysql-db mysqladmin ping -h"127.0.0.1" --silent; then
                        echo "✅ MySQL is ready!"
                        exit 0
                    fi
                    echo "⏳ Waiting for MySQL... ($i/20)"
                    sleep 5
                done
                echo "❌ MySQL did not start in time!"
                docker logs mysql-db
                exit 1
                '''
            }
        }

        stage('Run Spring Boot Container') {
            steps {
                echo "🚀 Starting Spring Boot container..."
                sh '''
                docker rm -f springboot-container || true
                docker run -d --name springboot-container \
                    --link mysql-db:mysql \
                    -p ${APP_PORT}:${APP_PORT} \
                    ${APP_NAME}:latest
                '''
            }
        }

        stage('Verify Spring Boot Health Endpoint') {
            steps {
                echo "🌐 Checking if the Spring Boot app is healthy..."
                sh '''
                sleep 20
                if curl -s http://localhost:${APP_PORT}/actuator/health | grep -q "UP"; then
                    echo "✅ Application is healthy!"
                else
                    echo "❌ Application health check failed!"
                    docker logs springboot-container
                    exit 1
                fi
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up containers..."
            sh '''
            docker stop springboot-container mysql-db || true
            docker rm springboot-container mysql-db || true
            '''
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check logs for details."
        }
    }
}

