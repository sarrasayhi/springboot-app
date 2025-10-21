pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app"
        DOCKERHUB_USER = "sarra784"
        CONTAINER_NAME = "springboot-app"
        MYSQL_CONTAINER = "mysql-db"
        MYSQL_IMAGE = "mysql:8.0"
        DB_NAME = "studentdb"
        DB_USER = "root"
        DB_PASSWORD = "root"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sarrasayhi/springboot-app.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKERHUB_PASS', usernameVariable: 'DOCKERHUB_USER')]) {
                        sh '''
                            echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                            docker tag $DOCKER_IMAGE $DOCKERHUB_USER/$DOCKER_IMAGE:latest
                            docker push $DOCKERHUB_USER/$DOCKER_IMAGE:latest
                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                        echo "🧩 Deploying application to Kubernetes..."
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        echo "✅ Application deployed successfully!"
                        kubectl get pods
                        kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build, Docker push, and Kubernetes deployment successful!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs.'
        }
    }
}

