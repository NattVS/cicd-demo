#!/usr/bin/env groovy

pipeline {
    agent any

    environment {
        APP_NAME = "cicd-demo"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/helderklemp/cicd-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${APP_NAME}:latest .'
            }
        }

        stage('Static Analysis (SonarQube)') {
            steps {
                sh '''
                mvn sonar:sonar \
                -Dsonar.projectKey=mi-app \
                -Dsonar.host.url=http://sonarqube:9000
                '''
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                sh 'trivy image ${APP_NAME}:latest'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker run -d -p 8080:8080 ${APP_NAME}:latest'
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
            cleanWs()
        }
    }
}