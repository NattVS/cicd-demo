#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        APP_NAME = "cicd-demo"
        DOCKER_HOST = "tcp://host.docker.internal:2375"
    }

    stages {

        // PUNTO 1: Etapas básicas del pipeline
        stage('Checkout') {
            steps {
                checkout scm  // Usa el repo configurado en el job de Jenkins
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                // -DforkCount=0: tests en mismo JVM (evita crash OOM en Docker)
                // -Djacoco.skip=true: desactiva agente JaCoCo que conflicta con forkCount=0
                sh 'mvn test -DforkCount=0 -Djacoco.skip=true'
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
