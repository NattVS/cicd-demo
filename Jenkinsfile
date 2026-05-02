#!/usr/bin/env groovy

pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        APP_NAME = "cicd-demo"
        SONAR_URL = "http://host.docker.internal:9000"
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

        // PUNTO 2 y 3.1: Etapas avanzadas de calidad y seguridad y Análisis Estático con Autenticación  

        stage('Static Analysis (SonarQube)') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                    mvn sonar:sonar -Dsonar.projectKey=cicd-demo
                    '''
                }
            }
        }

        // PUNTO 3.3: Puerta de Calidad (Gatekeeping)
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    // Detiene el pipeline si Sonarqube encuentra errores críticos
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // PUNTO 3.2: Escaneo de Seguridad
        stage('Container Security Scan (Trivy)') {
            steps {
                sh 'docker save ${APP_NAME}:latest -o mi-imagen.tar'
                sh 'trivy image --severity CRITICAL --timeout 30m --input mi-imagen.tar'
            }
        }

        // PUNTO 4.1: Despliegue en máquina local
        stage('Deploy') {
            steps {
                // Detiene cualquier contenedor anterior para que el puerto 80 no dé conflicto
                sh 'docker rm -f mi-app-container || true'
                sh 'docker run -d --name mi-app-container -p 80:8080 ${APP_NAME}:latest'
            }
        }
    }

    post {
        // PUNTO 3.4 y 4.2: Limpieza e Infraestructura/Notif
        always {
            echo 'Limpiando entorno...'
            cleanWs()
            // Elimina imgs huérfanas de Docker 
            sh 'docker image prune -f'
        }
        success {
            echo 'Pipeline completado con éxito. Aplicación desplegada.'
        }
        failure {
            echo 'Pipeline falló. Revisar logs de SonarQube o Trivy para más detalles.'
        }
    }
}