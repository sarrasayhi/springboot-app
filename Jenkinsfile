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
                sh 'sudo docker build -t $DOCKER_IMAGE .'
            }
        }

        stage('Run MySQL Container') {
            steps {
                script {
                    // Stop and remove old container if exists
                    sh 'sudo docker rm -f $MYSQL_CONTAINER || true'
                    sh '''
                        sudo docker run -d --name $MYSQL_CONTAINER \
                        -e MYSQL_ROOT_PASSWORD=$DB_PASSWORD \
                        -e MYSQL_DATABASE=$DB_NAME \
                        -p 3306:3306 $MYSQL_IMAGE
                    '''
                    // Wait for MySQL to start
                    sh 'sleep 20'
                }
            }
        }

        stage('Run Spring Boot App') {
            steps {
                script {
                    // Stop old app container if exists
                    sh 'sudo docker rm -f $CONTAINER_NAME || true'
                    // Run new container connected to MySQL
                    sh '''
                        sudo docker run -d --name $CONTAINER_NAME \
                        -e SPRING_DATASOURCE_URL=jdbc:mysql://$MYSQL_CONTAINER:3306/$DB_NAME \
                        -e SPRING_DATASOURCE_USERNAME=$DB_USER \
                        -e SPRING_DATASOURCE_PASSWORD=$DB_PASSWORD \
                        --link $MYSQL_CONTAINER:mysql \
                        -p 8080:8080 $DOCKER_IMAGE
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build and Deployment successful!'
            sh 'sudo docker ps'
        }
        failure {
            echo '❌ Build failed. Check logs.'
        }
    }
}

