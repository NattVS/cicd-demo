#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'Maven'  // Debe coincidir con el nombre en Jenkins > Tools
    }

    environment {
        APP_NAME = "cicd-demo"
    }

    stages {

        // PUNTO 1: Etapas básicas del pipeline
        stage('Checkout') {
            steps {
                git 'https://github.com/helderklemp/cicd-demo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${APP_NAME}:latest .'
            }
        }

        // PUNTO 2: Etapas avanzadas de calidad y seguridad
        stage('Static Analysis (SonarQube)') {
            steps {
                sh '''
                mvn sonar:sonar \
                -Dsonar.projectKey=cicd-demo \
                -Dsonar.host.url=http://sonarqube:9000
                '''
            }
        }

        stage('Container Security Scan (Trivy)') {
            steps {
                sh 'trivy image --exit-code 1 --severity CRITICAL ${APP_NAME}:latest'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker run -d -p 80:8080 ${APP_NAME}:latest'
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
            cleanWs()
        }
        failure {
            echo 'Pipeline falló. Revisar logs para más detalles.'
        }
    }
}
