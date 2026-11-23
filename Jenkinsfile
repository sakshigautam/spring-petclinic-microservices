
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
                bat '''
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
                bat '''
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

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis'
        
                withSonarQubeEnv('SonarQube1') {
                    bat '''
                        mvnw.cmd -B sonar:sonar ^
                        -Dsonar.projectKey=spring-petclinic-ci-project-scan^
                        -Dsonar.projectName=spring-petclinic-ci-project-scan
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images for microservices'
                bat '''
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
