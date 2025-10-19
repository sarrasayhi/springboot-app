pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app:latest"
        DOCKER_NETWORK = "spring-net"
        SONARQUBE_ENV = "SonarQube"  // Must match the name in Jenkins SonarQube config
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build & Test with Maven') {
            steps {
                echo 'Building and testing Spring Boot app...'
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Code Analysis') {
            environment {
                scannerHome = tool 'SonarQubeScanner' // Must match Jenkins tool name
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    echo 'Running SonarQube analysis...'
                    sh '''
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=springboot-app \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target
                    '''
                }
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

                    // Create Docker network if it doesn’t exist
                    sh 'docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || docker network create ${DOCKER_NETWORK}'

                    // Stop and remove existing containers
                    sh '''
                        docker ps -q --filter "name=springboot-container" | grep -q . && docker stop springboot-container && docker rm springboot-container || true
                        docker ps -q --filter "name=mysql-db" | grep -q . && docker stop mysql-db && docker rm mysql-db || true
                    '''

                    // Run MySQL
                    sh '''
                        docker run -d --name mysql-db \
                        --network ${DOCKER_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=root \
                        -e MYSQL_DATABASE=studentdb \
                        -p 3306:3306 \
                        mysql:8
                    '''

                    // Wait for MySQL
                    sh 'sleep 25'

                    // Run Spring Boot container
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
            echo '✅ Build, Tests, SonarQube Analysis, and Deployment completed successfully!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for errors.'
        }
    }
}

