pipeline {
    agent any

    environment {
        IMAGE_NAME = 'spring-app'
        DOCKER_NETWORK = 'spring-net'
        MYSQL_CONTAINER = 'mysql-db'
        MYSQL_ROOT_PASSWORD = 'root'
        MYSQL_DATABASE = 'studentdb'
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo '📦 Cloning source code from GitHub...'
                checkout scm
            }
        }

        stage('Start MySQL Container') {
            steps {
                script {
                    echo '🛢 Starting local MySQL container...'

                    // Ensure Docker network exists
                    sh '''
                    docker network create ${DOCKER_NETWORK} || true
                    '''

                    // Remove old MySQL container if exists
                    sh '''
                    docker rm -f ${MYSQL_CONTAINER} || true
                    '''

                    // Run MySQL container
                    sh '''
                    docker run -d --name ${MYSQL_CONTAINER} \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} \
                        -e MYSQL_DATABASE=${MYSQL_DATABASE} \
                        -p 3306:3306 mysql:8.0
                    '''

                    // Wait for MySQL to become ready
                    sh '''
                    echo "⏳ Waiting for MySQL to start..."
                    for i in {1..40}; do
                        if docker exec ${MYSQL_CONTAINER} mysqladmin ping -uroot -p${MYSQL_ROOT_PASSWORD} --silent; then
                            echo "✅ MySQL is ready!"
                            exit 0
                        fi
                        echo "⌛ Waiting... ($i/40)"
                        sleep 5
                    done
                    echo "❌ MySQL failed to start"
                    docker logs ${MYSQL_CONTAINER}
                    exit 1
                    '''
                }
            }
        }

        stage('Build & Test with Maven') {
            steps {
                echo '⚙️ Building project and running tests...'
                sh 'mvn clean package -DskipTests=false'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONARQUBE = credentials('sonar-token') // Jenkins Sonar token credential
            }
            steps {
                withSonarQubeEnv('sonar-server') { // replace 'sonar-server' with your configured server name
                    sh '''
                    echo "🔍 Running SonarQube analysis..."
                    mvn sonar:sonar \
                        -Dsonar.projectKey=spring-app \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONARQUBE
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '🐳 Building local Docker image for Spring Boot app...'
                sh '''
                docker build -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Deploy Locally') {
            steps {
                echo '🚀 Deploying Spring Boot container connected to MySQL...'
                sh '''
                docker rm -f spring-app || true
                docker run -d --name spring-app \
                    --network ${DOCKER_NETWORK} \
                    -e SPRING_DATASOURCE_URL=jdbc:mysql://${MYSQL_CONTAINER}:3306/${MYSQL_DATABASE} \
                    -e SPRING_DATASOURCE_USERNAME=root \
                    -e SPRING_DATASOURCE_PASSWORD=root \
                    -p 8081:8081 ${IMAGE_NAME}:latest
                '''
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up after pipeline...'
            sh '''
            docker ps -a
            '''
        }
        success {
            echo '✅ Pipeline finished successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs for details.'
        }
    }
}

