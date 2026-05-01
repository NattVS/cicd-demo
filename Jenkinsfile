#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'Maven'
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
                // -DforkCount=0 evita que Surefire lance un JVM separado
                // (soluciona crash por falta de memoria en Docker)
                sh 'mvn test -DforkCount=0'
            }
            post {
                failure {
                    echo 'Tests fallaron - revisar reporte en target/surefire-reports'
                }
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
