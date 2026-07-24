pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_NAME = 'expensesapp'
        REPO_URL = 'https://github.com/aadityapalsagwan/java-springboot-cicd-jenkins-docker.git'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository...'
                git url: "${REPO_URL}", branch: 'main'
            }
        }
        
        stage('Explore Repository Structure') {
            steps {
                echo 'Checking repository structure...'
                sh '''
                    echo "=== Current Directory ==="
                    pwd
                    echo "=== Listing all files ==="
                    find . -type f -name "*.java" -o -name "pom.xml" -o -name "*.yml" -o -name "*.yaml" -o -name "Dockerfile" | head -20
                    echo "=== Directory structure ==="
                    ls -la
                    echo "=== Checking for pom.xml ==="
                    find . -name "pom.xml" 2>/dev/null
                '''
            }
        }
        
        stage('Clean Up Existing Files') {
            steps {
                echo 'Removing existing Docker configuration files...'
                sh '''
                    # Find and remove existing Docker configuration files
                    find . -name "Dockerfile" -o -name "docker-compose.yml" -o -name "docker-compose.yaml" | xargs rm -f 2>/dev/null || true
                    
                    echo "=== Files after cleanup ==="
                    ls -la
                '''
            }
        }
        
        stage('Create Dockerfile') {
            steps {
                echo 'Creating Dockerfile...'
                script {
                    // First, let's find where pom.xml is located
                    def pomLocation = sh(script: 'find . -name "pom.xml" -type f | head -1', returnStdout: true).trim()
                    echo "Found pom.xml at: ${pomLocation}"
                    
                    // Create Dockerfile at the root of workspace
                    writeFile file: 'Dockerfile', text: '''# stage 1 – Build the JAR (Java app runtime) using maven

FROM maven:3.8.3-openjdk-17 AS builder

WORKDIR /app

# Copy pom.xml first for better layer caching
COPY pom.xml .
# Download dependencies
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# stage 2 – Create runtime image
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copy the built jar from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Expose port
EXPOSE 8080

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]'''
                }
            }
        }
        
        stage('Create docker-compose.yml') {
            steps {
                echo 'Creating docker-compose.yml...'
                writeFile file: 'docker-compose.yml', text: '''version: '3.8'

services:
  mysql_db:
    image: mysql:8.0
    container_name: mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: Test@123
      MYSQL_DATABASE: expenses_tracker
      MYSQL_USER: app_user
      MYSQL_PASSWORD: App@123
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-pTest@123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  java_app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: expenses-app
    depends_on:
      mysql_db:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-db:3306/expenses_tracker?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: Test@123
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
      SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT: org.hibernate.dialect.MySQL8Dialect
    ports:
      - "80:8080"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  mysql_data:'''
            }
        }
        
        stage('Verify Files') {
            steps {
                echo 'Verifying created files...'
                sh '''
                    echo "=== Current directory ==="
                    pwd
                    echo "=== Listing files ==="
                    ls -la
                    echo "=== Dockerfile content (first 20 lines) ==="
                    head -20 Dockerfile || echo "Dockerfile not found"
                    echo "=== docker-compose.yml content (first 20 lines) ==="
                    head -20 docker-compose.yml || echo "docker-compose.yml not found"
                    echo "=== Checking for pom.xml ==="
                    find . -name "pom.xml" 2>/dev/null
                    if [ -f "pom.xml" ]; then
                        echo "pom.xml exists in current directory"
                        ls -la pom.xml
                    else
                        echo "Looking for pom.xml in subdirectories..."
                        find . -name "pom.xml" -type f | head -5
                    fi
                '''
            }
        }
        
        stage('Build and Deploy') {
            steps {
                echo 'Building and deploying application...'
                sh '''
                    echo "=== Building Docker image ==="
                    docker build --no-cache -t expensesapp .
                    
                    echo "=== Checking built image ==="
                    docker images | grep expensesapp
                    
                    echo "=== Stopping any existing containers ==="
                    docker compose down || true
                    
                    echo "=== Starting application ==="
                    docker compose up -d
                    
                    echo "=== Waiting for services to start ==="
                    sleep 10
                    
                    echo "=== Checking container status ==="
                    docker compose ps
                    
                    echo "=== Checking logs ==="
                    docker compose logs --tail=10
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployment...'
                script {
                    sleep(30) // Give services time to start
                    
                    sh '''
                        echo "=== Final container status ==="
                        docker compose ps
                        
                        echo "=== Checking application health ==="
                        echo "Waiting for application to be ready..."
                        # Try multiple endpoints
                        for i in {1..10}; do
                            if curl -f http://localhost:80/actuator/health 2>/dev/null; then
                                echo "Application is healthy!"
                                break
                            elif curl -f http://localhost:80 2>/dev/null; then
                                echo "Application is responding!"
                                break
                            else
                                echo "Attempt $i: Application not ready yet..."
                                sleep 10
                            fi
                        done
                        
                        echo "=== Application endpoints ==="
                        echo "Java App: http://localhost:80"
                        echo "MySQL: localhost:3306"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed. Displaying final status...'
            sh '''
                echo "=== Final Docker containers ==="
                docker ps -a
                
                echo "=== Application logs ==="
                docker compose logs --tail=50 || true
                
                echo "=== Docker images ==="
                docker images | grep -E "(expensesapp|mysql)"
            '''
        }
        success {
            echo 'Deployment completed successfully!'
            echo 'Access the application at: http://localhost:80'
        }
        failure {
            echo 'Deployment failed!'
            sh '''
                echo "=== Debug information ==="
                echo "Docker version:"
                docker --version
                echo "Docker Compose version:"
                docker compose version
                echo "Current directory:"
                pwd
                echo "Directory contents:"
                ls -la
                echo "Checking for source files:"
                find . -name "*.java" | head -5
                echo "Full error logs:"
                docker compose logs || true
            '''
        }
    }
}