pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "springboot-app"
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

        stage('Run MySQL Container') {
            steps {
                script {
                    sh '''
                        if [ "$(docker ps -aq -f name=$MYSQL_CONTAINER)" ]; then
                            echo "Stopping and removing existing $MYSQL_CONTAINER..."
                            docker stop $MYSQL_CONTAINER || true
                            docker rm -f $MYSQL_CONTAINER || true
                        fi

                        docker run -d --name $MYSQL_CONTAINER \
                            -e MYSQL_ROOT_PASSWORD=$DB_PASSWORD \
                            -e MYSQL_DATABASE=$DB_NAME \
                            -p 3306:3306 $MYSQL_IMAGE

                        echo "Waiting for MySQL to start..."
                        sleep 20
                    '''
                }
            }
        }

        stage('Run Spring Boot App') {
            steps {
                script {
                    sh '''
                        if [ "$(docker ps -aq -f name=$CONTAINER_NAME)" ]; then
                            echo "Stopping and removing existing $CONTAINER_NAME..."
                            docker stop $CONTAINER_NAME || true
                            docker rm -f $CONTAINER_NAME || true
                        fi

                        docker run -d --name $CONTAINER_NAME \
                            -e SPRING_DATASOURCE_URL=jdbc:mysql://$MYSQL_CONTAINER:3306/$DB_NAME \
                            -e SPRING_DATASOURCE_USERNAME=$DB_USER \
                            -e SPRING_DATASOURCE_PASSWORD=$DB_PASSWORD \
                            --link $MYSQL_CONTAINER:mysql \
                            -p 8081:8081 $DOCKER_IMAGE
                    '''
                }
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "🔐 Logging in to DockerHub..."
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag $DOCKER_IMAGE $DOCKER_USER/$DOCKER_IMAGE:latest
                        docker push $DOCKER_USER/$DOCKER_IMAGE:latest
                        echo "✅ Image pushed successfully to DockerHub!"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build and Deployment successful!'
            sh 'docker ps'
        }
        failure {
            echo '❌ Build failed. Check logs.'
        }
    }
}

