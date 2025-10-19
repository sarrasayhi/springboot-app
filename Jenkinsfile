pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build Maven Project') {
            steps {
                echo 'Building Spring Boot app with Maven...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${DOCKER_IMAGE} .'
            }
        }

        stage('Run Containers') {
            steps {
                script {
                    echo 'Setting up Docker network and containers...'

                    // Create the Docker network if it doesn’t exist
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Remove old containers if they exist
                    sh 'docker rm -f mysql-db springboot-container || true'

                    // Start MySQL container
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8
                    '''

                    // Wait for MySQL to start up
                    sh 'sleep 25'

                    // Start Spring Boot container
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
            echo '✅ Spring Boot app built and running on Docker network spring-net!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for errors.'
        }
    }
}

