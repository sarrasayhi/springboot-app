pipeline {
    agent any

    tools {
        maven 'Maven3'       // Jenkins Maven installation name
        jdk 'jdk17'          // Jenkins JDK installation name
    }

    environment {
        IMAGE_NAME = 'springboot-app'
        SONARQUBE = 'SonarQube'   // Jenkins SonarQube server name
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                echo 'Cloning repository...'
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build with Maven') {
            steps {
                echo 'Building Spring Boot app with Maven...'
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                      mvn sonar:sonar \
                        -Dsonar.projectKey=springboot-app \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t $IMAGE_NAME:latest .'
            }
        }

        stage('Run Docker Container') {
            steps {
                echo 'Running the Docker container...'
                sh 'docker stop springboot-container || true && docker rm springboot-container || true'
                sh 'docker run -d --name springboot-container -p 9090:8081 $IMAGE_NAME:latest'
            }
        }
    }

    post {
        success {
            echo '✅ Spring Boot app built, analyzed, and deployed in Docker successfully!'
        }
        failure {
            echo '❌ Build failed! Check the logs for details.'
        }
    }
}

