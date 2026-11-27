
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
        
                withSonarQubeEnv('MySonarQube') {
                    bat '''
                        mvnw.cmd -B install ^
                        org.sonarsource.scanner.maven:sonar-maven-plugin:5.9.0.7382:sonar ^
                        -Dsonar.projectKey=spring-petclinic-ci-project-scan^
                        -Dsonar.projectName=spring-petclinic-ci-project-scan^
                    '''
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

       
        stage('Trivy Scan Images') {
            steps {
                script {
                    echo 'Discovering PetClinic Docker images'

                    bat '''
                        docker images --format "{{.Repository}}:{{.Tag}}" > petclinic-images.txt
                    '''

                    def images = readFile('petclinic-images.txt')
                        .readLines()
                        .collect { it.trim() }
                        .findAll {
                            it &&
                            it.toLowerCase().contains('petclinic') &&
                            !it.contains('<none>')
                        }
                        .unique()

                    if (images.isEmpty()) {
                        error('No PetClinic Docker images found. Check image names and tags.')
                    }

                    echo "Found ${images.size()} PetClinic images:"
                    images.each { echo " - ${it}" }

                    def failedImages = []

                    for (imageName in images) {
                        def reportName = imageName
                            .replaceAll('[^A-Za-z0-9_.-]', '_') +
                            '_trivy_scan.txt'

                        echo "Scanning image: ${imageName}"

                        def scanStatus = bat(
                            returnStatus: true,
                            script: """
                                @echo off
                                trivy.exe image ^
                                --exit-code 1 ^
                                --severity CRITICAL,HIGH ^
                                --format table ^
                                --output "${reportName}" ^
                                "${imageName}"
                            """
                        )

                        if (fileExists(reportName)) {
                            echo "Trivy report for ${imageName}:"
                            bat "type \"${reportName}\""
                        }

                        if (scanStatus != 0) {
                            failedImages.add(imageName)
                            echo "Trivy scan failed or vulnerabilities were found: ${imageName}"
                        } else {
                            echo "Trivy scan passed: ${imageName}"
                        }
                    }

                    archiveArtifacts(
                        artifacts: '*_trivy_scan.txt',
                        allowEmptyArchive: true
                    )

                    if (!failedImages.isEmpty()) {
                        error("Trivy detected vulnerabilities or scan errors in: ${failedImages.join(', ')}")
                    }
                }
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
