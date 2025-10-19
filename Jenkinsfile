pipeline {
    agent any

    environment {
        DOCKER_NETWORK = "spring-net"
        APP_IMAGE = "springboot-student-app"
        MYSQL_CONTAINER = "mysql-db"
        MYSQL_ROOT_PASSWORD = "root"
        MYSQL_DATABASE = "studentdb"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "📦 Checking out source code..."
                checkout scm
            }
        }

        stage('Build JAR (skip tests)') {
            steps {
                echo "⚙️ Building Spring Boot JAR (tests skipped)"
                sh 'mvn clean package -DskipTests=true'
            }
        }

        stage('Docker Compose Up') {
            steps {
                echo "🧱 Starting containers with Docker Compose v2"
                sh '''
                    docker network create ${DOCKER_NETWORK} || true
                    docker compose down || true
                    docker compose up -d --build
                '''
            }
        }

        stage('Verify MySQL & App Startup') {
            steps {
                script {
                    echo "⏳ Waiting for MySQL to be ready..."
                    sh '''
                        sleep 20
                        for i in {1..30}; do
                            if docker exec ${MYSQL_CONTAINER} mysql -u${MYSQL_ROOT_PASSWORD} -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1" > /dev/null 2>&1; then
                                echo "✅ MySQL is ready!"
                                break
                            fi
                            echo "⏳ MySQL not ready yet ($i/30)..."
                            sleep 3
                        done
                    '''
                    echo "🔍 Checking if Spring Boot app is running..."
                    sh '''
                        sleep 15
                        if docker ps | grep -q ${APP_IMAGE}; then
                            echo "✅ Spring Boot app container is running!"
                        else
                            echo "❌ Spring Boot app container is not running!"
                            docker compose logs
                            exit 1
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "🧹 Cleaning up containers..."
            sh '''
                docker compose down || true
                docker network rm ${DOCKER_NETWORK} || true
            '''
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check logs for details."
            sh 'docker compose logs || true'
        }
    }
}

