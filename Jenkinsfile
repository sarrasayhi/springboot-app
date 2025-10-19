pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Spring Boot App') {
            steps {
                sh '''
                    echo "🚀 Building Spring Boot JAR (tests skipped)"
                    mvn clean package -DskipTests=true
                '''
            }
        }

        stage('Docker Compose Up') {
            steps {
                sh '''
                    echo "🧱 Starting containers with docker-compose"
                    docker-compose down || true
                    docker-compose up -d --build
                    echo "⏳ Waiting for MySQL to become healthy..."
                    sleep 25
                    docker ps
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "🔍 Checking running containers"
                    docker logs springboot-app --tail 20
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
            sh 'docker-compose logs mysql-db || true'
        }
    }
}

