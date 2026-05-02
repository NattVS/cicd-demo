# CICD-DEMO

Proyecto de demostración para la construcción de un pipeline CI/CD completo usando **Jenkins**, **Docker**, **SonarQube** y **Trivy**. La aplicación base es un servicio REST en **Spring Boot**.



## 1. Levantar los servicios

### Jenkins

```bash
docker run -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

> **En Windows:** Docker Desktop debe tener habilitada la opción **"Expose daemon on tcp://localhost:2375 without TLS"** en Settings → General. Esto permite que Jenkins se conecte a Docker vía TCP.

Luego, instalar Docker CLI dentro del contenedor de Jenkins (requerido para el stage Docker Build):

```bash
docker exec -u root $(docker ps -q --filter ancestor=jenkins/jenkins:lts) \
  bash -c "apt-get update -q && apt-get install -y docker.io"
```

### SonarQube

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
```

Accede en `http://localhost:9000` (usuario: `admin`, contraseña: `admin`). Luego genera un **token de autenticación** en My Account → Security.


## 2. Configurar Jenkins

### Plugins requeridos

Instalar desde **Manage Jenkins → Plugins**:

- Git
- Pipeline
- Docker Pipeline
- SonarQube Scanner for Jenkins

### Maven

En **Manage Jenkins → Tools → Maven installations**:
- Name: `Maven`
- Install automatically — versión `3.9.6`

### Credencial de SonarQube

En **Manage Jenkins → Credentials → Global → Add Credentials**:
- Kind: `Secret text`
- Secret: (token generado en SonarQube)
- ID: `sonar-token`

### Servidor de SonarQube

En **Manage Jenkins → System → SonarQube installations**:
- Name: `sonarqube-server`
- URL: `http://host.docker.internal:9000`
- Token: seleccionar credencial `sonar-token`

### Webhook de SonarQube → Jenkins

En SonarQube, ir a **Project Settings → Webhooks → Create**:
- URL: `http://host.docker.internal:8080/sonarqube-webhook/`

Esto permite que Jenkins espere el Quality Gate con `waitForQualityGate`.

### Instalar Trivy en Jenkins

```bash
docker exec -u root $(docker ps -q --filter ancestor=jenkins/jenkins:lts) bash -c \
  "apt-get install -y wget apt-transport-https gnupg && \
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - && \
   echo 'deb https://aquasecurity.github.io/trivy-repo/deb generic main' \
   > /etc/apt/sources.list.d/trivy.list && \
   apt-get update && apt-get install -y trivy"
```

## 3. Crear el Job en Jenkins

1. **New Item** → nombre: `cicd-demo` → tipo: **Pipeline**
2. En la sección **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<tu-usuario>/cicd-demo.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. En **Build Triggers**: activar **Poll SCM** con expresión `* * * * *` para detección automática de commits
4. Guardar y ejecutar **Build Now**


## 4. Descripción del Pipeline

El pipeline declarativo (`Jenkinsfile`) ejecuta las siguientes etapas en orden:

| Stage | Descripción |
|---|---|
| **Checkout** | Clona el repositorio configurado en el job (`checkout scm`) |
| **Build** | Compila la aplicación con `mvn clean package -DskipTests` |
| **Test** | Corre los tests con `mvn test -DforkCount=0 -Djacoco.skip=true` |
| **Docker Build** | Construye la imagen Docker `cicd-demo:latest` |
| **Static Analysis** | Analiza el código con SonarQube vía `mvn sonar:sonar` |
| **Quality Gate** | Espera el resultado de SonarQube; aborta si falla |
| **Container Security Scan** | Escanea la imagen con Trivy buscando vulnerabilidades CRITICAL |
| **Deploy** | Despliega el contenedor en el puerto 80 (solo si branch es `main`) |

El bloque `post` limpia el workspace (`cleanWs()`) y las imágenes huérfanas (`docker image prune -f`) al finalizar cada ejecución, independientemente del resultado.

## 5. Variables de entorno del Jenkinsfile

| Variable | Valor | Descripción |
|---|---|---|
| `APP_NAME` | `cicd-demo` | Nombre de la imagen Docker |
| `DOCKER_HOST` | `tcp://host.docker.internal:2375` | Conexión al daemon Docker del host desde el contenedor Jenkins |

## 6. Comportamiento del Gatekeeping

- **Quality Gate (SonarQube):** Si se detecta un Security Hotspot, el pipeline se aborta antes del deploy.
- **Trivy:** Si se detectan vulnerabilidades `CRITICAL`, el pipeline falla con exit code 1. Para omitir temporalmente este bloqueo (ej. pruebas de deploy), retirar el flag `--exit-code 1` del stage `Container Security Scan`.


## Estructura del proyecto

```
cicd-demo/
├── Jenkinsfile          # Pipeline declarativo CI/CD
├── Dockerfile           # Imagen base: eclipse-temurin:17-jre-alpine
├── pom.xml              # Configuración Maven (Spring Boot 2.x)
├── src/                 # Código fuente Java
└── README.md            # Este archivo
```
