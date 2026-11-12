
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out source code'
                checkout scm
            }
        }

        stage('Check Environment') {
            steps {
                sh '''
                    java -version
                    git --version
                    docker --version
                    docker info
                '''
            }
        }

        stage('Build and Test') {
            steps {
                echo 'Building application using Maven'
                sh '''
                    chmod +x mvnw
                    ./mvnw -B clean verify
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images for microservices'
                sh '''
                    ./mvnw -B install -DskipTests \
                        -P buildDocker
                '''
            }
        }
    }

    post {
        success {
            echo 'CI Pipeline completed successfully!'
        }

        failure {
            echo 'CI Pipeline failed. Please check the logs.'
        }

        always {
            echo 'CI Pipeline execution finished.'
        }
    }
}
