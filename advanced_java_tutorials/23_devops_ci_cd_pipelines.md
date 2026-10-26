# Advanced DevOps and CI/CD Pipelines

## Table of Contents
1. [Introduction to DevOps and CI/CD](#introduction-to-devops-and-cicd)
2. [Pipeline Architecture Fundamentals](#pipeline-architecture-fundamentals)
3. [Popular CI/CD Tools and Platforms](#popular-cicd-tools-and-platforms)
4. [Infrastructure as Code (IaC)](#infrastructure-as-code-iac)
5. [Containerization and Orchestration](#containerization-and-orchestration)
6. [Deployment Strategies](#deployment-strategies)
7. [Monitoring and Observability](#monitoring-and-observability)
8. [Security and Compliance Automation](#security-and-compliance-automation)
9. [Performance Optimization](#performance-optimization)
10. [Best Practices and Anti-Patterns](#best-practices-and-anti-patterns)

## Introduction to DevOps and CI/CD

DevOps represents a cultural and technical movement that bridges the gap between development and operations teams, emphasizing collaboration, automation, and continuous improvement. CI/CD (Continuous Integration/Continuous Delivery) forms the technical backbone of modern software delivery pipelines.

### Core Principles

1. **Automation First**: Eliminate manual processes through comprehensive automation
2. **Continuous Feedback**: Implement rapid feedback loops throughout the development lifecycle
3. **Shared Responsibility**: Foster collaboration between development, testing, and operations
4. **Fail Fast, Recover Faster**: Embrace controlled failures with rapid recovery mechanisms
5. **Measurement and Monitoring**: Quantify everything to drive data-driven decisions

### Benefits in Enterprise Environments

```java
// Example: Health check endpoint for monitoring pipeline status
@RestController
public class PipelineHealthController {
    
    @Autowired
    private PipelineStatusService pipelineStatusService;
    
    @GetMapping("/health/pipeline")
    public ResponseEntity<PipelineHealth> getPipelineHealth() {
        PipelineHealth health = pipelineStatusService.getCurrentHealth();
        HttpStatus status = health.isHealthy() ? HttpStatus.OK : HttpStatus.SERVICE_UNAVAILABLE;
        return ResponseEntity.status(status).body(health);
    }
}
```

## Pipeline Architecture Fundamentals

Modern CI/CD pipelines follow a multi-stage architecture with distinct phases that ensure quality, security, and reliability.

### Pipeline Stages

1. **Source Control Management**: Code commit, branch management, and pull requests
2. **Build and Compilation**: Code compilation, dependency resolution, and artifact creation
3. **Automated Testing**: Unit, integration, and contract testing
4. **Security Scanning**: Static analysis, vulnerability detection, and compliance checks
5. **Deployment**: Artifact deployment to various environments
6. **Validation and Monitoring**: Post-deployment verification and health checks

### Pipeline Execution Engines

```yaml
# Example: GitHub Actions workflow with parallel execution
name: CI/CD Pipeline
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: [ 17, 21 ]
    steps:
    - uses: actions/checkout@v3
    - name: Setup Java
      uses: actions/setup-java@v3
      with:
        java-version: ${{ matrix.java-version }}
        distribution: 'temurin'
    - name: Cache Maven dependencies
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    - name: Build with Maven
      run: mvn clean install -DskipTests
    - name: Run Tests
      run: mvn test
```

## Popular CI/CD Tools and Platforms

### Jenkins

Jenkins remains a popular choice for complex pipeline orchestration with extensive plugin ecosystem.

```groovy
// Jenkinsfile for multi-stage pipeline
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.1'
        jdk 'OpenJDK 17'
    }
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        DEPLOY_ENV = 'staging'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/example/app.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn verify -DskipUnitTests'
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'mvn dependency-check:check'
            }
            post {
                failure {
                    script {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_REGISTRY}/myapp:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                    sh "docker push ${imageTag}"
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    deployToKubernetes(env.DEPLOY_ENV)
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend channel: '#deployments', 
                      message: "Build ${env.BUILD_NUMBER} succeeded for ${env.JOB_NAME}"
        }
        failure {
            slackSend channel: '#deployments', 
                      message: "Build ${env.BUILD_NUMBER} failed for ${env.JOB_NAME}"
        }
    }
}

def deployToKubernetes(environment) {
    kubernetesDeploy(
        configs: 'k8s/deployment.yaml',
        kubeconfigId: 'kubeconfig',
        environment: environment
    )
}
```

### GitLab CI/CD

GitLab provides integrated CI/CD with tight repository integration.

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

before_script:
  - echo "Starting pipeline for commit $CI_COMMIT_SHA"
  - export MAVEN_CLI_OPTS="--batch-mode --errors --fail-at-end --show-version"

build:
  stage: build
  image: maven:3.8-openjdk-17
  script:
    - mvn $MAVEN_CLI_OPTS compile
    - mvn $MAVEN_CLI_OPTS package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week
  cache:
    key: maven
    paths:
      - .m2/repository/

unit_tests:
  stage: test
  image: maven:3.8-openjdk-17
  script:
    - mvn $MAVEN_CLI_OPTS test
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

integration_tests:
  stage: test
  image: maven:3.8-openjdk-17
  services:
    - postgres:13
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
  script:
    - mvn $MAVEN_CLI_OPTS verify -DskipUnitTests
  artifacts:
    reports:
      junit: target/failsafe-reports/TEST-*.xml

security_scan:
  stage: security
  image: maven:3.8-openjdk-17
  script:
    - mvn $MAVEN_CLI_OPTS dependency-check:check
  artifacts:
    reports:
      dependency_scanning: dependency-check-report.json

build_image:
  stage: deploy
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  only:
    - main

deploy_staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - |
      cat <<EOF | kubectl apply -f -
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: myapp
        template:
          metadata:
            labels:
              app: myapp
          spec:
            containers:
            - name: myapp
              image: $IMAGE_NAME
              ports:
              - containerPort: 8080
    EOF
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main
```

## Infrastructure as Code (IaC)

Infrastructure as Code enables version-controlled, repeatable infrastructure provisioning.

### Terraform for Cloud Infrastructure

```hcl
# main.tf - Example Terraform configuration for AWS
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC Configuration
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "main-vpc"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# Security Groups
resource "aws_security_group" "app_sg" {
  name        = "app-security-group"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-sg"
  }
}

# EC2 Instances with Auto Scaling
resource "aws_launch_template" "app" {
  name_prefix   = "app-server"
  image_id      = var.ami_id
  instance_type = var.instance_type

  user_data = base64encode(templatefile("${path.module}/user-data.sh", {
    app_version = var.app_version
  }))

  vpc_security_group_ids = [aws_security_group.app_sg.id]

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "app-server"
    }
  }
}

resource "aws_autoscaling_group" "app" {
  desired_capacity     = 2
  max_size             = 10
  min_size             = 2
  vpc_zone_identifier  = aws_subnet.public[*].id
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }

  target_group_arns = [aws_lb_target_group.app.arn]
}

# Load Balancer
resource "aws_lb" "app" {
  name               = "app-load-balancer"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.app_sg.id]
  subnets            = aws_subnet.public[*].id

  tags = {
    Name = "app-alb"
  }
}

resource "aws_lb_target_group" "app" {
  name     = "app-target-group"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "admin_cidr" {
  description = "CIDR block for administrative access"
  type        = string
}

variable "app_version" {
  description = "Application version to deploy"
  type        = string
}

# Outputs
output "load_balancer_dns" {
  value = aws_lb.app.dns_name
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

### Kubernetes Manifests

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: registry.example.com/myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Containerization and Orchestration

Containerization provides consistency across environments and enables scalable deployments.

### Docker Best Practices

```dockerfile
# Multi-stage Dockerfile for Java application
# Build stage
FROM maven:3.8.6-openjdk-17 AS builder

WORKDIR /app

# Copy dependency files first for better caching
COPY pom.xml .
COPY src/main/resources/application.properties src/main/resources/

# Download dependencies
RUN mvn dependency:go-offline -B

# Copy source code
COPY src ./src

# Build the application
RUN mvn clean package -DskipTests

# Runtime stage
FROM openjdk:17-jdk-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy the JAR file from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Create volume for temporary files
VOLUME /tmp

# Change ownership to non-root user
RUN chown appuser:appuser app.jar
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:postgresql://db:5432/myapp
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs

  db:
    image: postgres:14
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  prometheus:
    image: prom/prometheus:v2.37.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - app-network

  grafana:
    image: grafana/grafana-enterprise
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app-network
    depends_on:
      - prometheus

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:

networks:
  app-network:
    driver: bridge
```

## Deployment Strategies

Different deployment strategies provide various trade-offs between risk, downtime, and complexity.

### Blue-Green Deployment

```java
// DeploymentController.java - Managing deployment strategies
@RestController
@RequestMapping("/api/deploy")
public class DeploymentController {
    
    @Autowired
    private DeploymentService deploymentService;
    
    @PostMapping("/blue-green")
    public ResponseEntity<DeploymentResult> blueGreenDeploy(@RequestBody DeploymentRequest request) {
        try {
            DeploymentResult result = deploymentService.executeBlueGreenDeployment(request);
            return ResponseEntity.ok(result);
        } catch (DeploymentException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new DeploymentResult(false, e.getMessage()));
        }
    }
    
    @PostMapping("/canary")
    public ResponseEntity<DeploymentResult> canaryDeploy(@RequestBody CanaryDeploymentRequest request) {
        try {
            DeploymentResult result = deploymentService.executeCanaryDeployment(request);
            return ResponseEntity.ok(result);
        } catch (DeploymentException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new DeploymentResult(false, e.getMessage()));
        }
    }
}

@Service
public class DeploymentService {
    
    private static final Logger logger = LoggerFactory.getLogger(DeploymentService.class);
    
    @Autowired
    private KubernetesClient k8sClient;
    
    @Autowired
    private LoadBalancerClient lbClient;
    
    public DeploymentResult executeBlueGreenDeployment(DeploymentRequest request) 
            throws DeploymentException {
        try {
            // 1. Deploy new version to green environment
            String greenDeploymentName = deployToGreenEnvironment(request);
            
            // 2. Run health checks on green environment
            if (!isHealthy(greenDeploymentName)) {
                rollbackGreenDeployment(greenDeploymentName);
                throw new DeploymentException("Health checks failed for green deployment");
            }
            
            // 3. Switch traffic from blue to green
            switchTraffic("blue", "green");
            
            // 4. Verify traffic switch was successful
            if (!verifyTrafficSwitch("green")) {
                // Rollback traffic switch
                switchTraffic("green", "blue");
                throw new DeploymentException("Traffic switch verification failed");
            }
            
            // 5. Decommission blue environment (optional)
            if (request.isDecommissionBlue()) {
                decommissionBlueEnvironment();
            }
            
            logger.info("Blue-green deployment completed successfully");
            return new DeploymentResult(true, "Deployment successful");
            
        } catch (Exception e) {
            logger.error("Blue-green deployment failed", e);
            throw new DeploymentException("Deployment failed: " + e.getMessage(), e);
        }
    }
    
    private String deployToGreenEnvironment(DeploymentRequest request) {
        // Implementation for deploying to green environment
        // This would typically involve creating new Kubernetes resources
        // or updating existing ones with the new version
        return "green-deployment-" + System.currentTimeMillis();
    }
    
    private boolean isHealthy(String deploymentName) {
        // Implementation for health checking
        // This would typically involve checking Kubernetes readiness probes
        // and making sample requests to the service
        return true; // Simplified for example
    }
    
    private void rollbackGreenDeployment(String deploymentName) {
        // Implementation for rolling back green deployment
        logger.info("Rolling back green deployment: {}", deploymentName);
    }
    
    private void switchTraffic(String fromEnvironment, String toEnvironment) 
            throws DeploymentException {
        try {
            // Update load balancer configuration to switch traffic
            lbClient.updateRouting(fromEnvironment, toEnvironment);
            logger.info("Traffic switched from {} to {}", fromEnvironment, toEnvironment);
        } catch (Exception e) {
            throw new DeploymentException("Failed to switch traffic: " + e.getMessage(), e);
        }
    }
    
    private boolean verifyTrafficSwitch(String targetEnvironment) {
        // Implementation for verifying traffic switch
        // This might involve sending test requests and verifying responses
        return true; // Simplified for example
    }
    
    private void decommissionBlueEnvironment() {
        // Implementation for decommissioning blue environment
        logger.info("Decommissioning blue environment");
    }
}
```

### Canary Deployment

```java
// CanaryDeploymentService.java - Implementing canary deployment strategy
@Service
public class CanaryDeploymentService {
    
    private static final Logger logger = LoggerFactory.getLogger(CanaryDeploymentService.class);
    
    @Autowired
    private KubernetesClient k8sClient;
    
    @Autowired
    private MetricsService metricsService;
    
    public DeploymentResult executeCanaryDeployment(CanaryDeploymentRequest request) 
            throws DeploymentException {
        try {
            // 1. Deploy canary version with small traffic percentage
            String canaryDeployment = deployCanary(request);
            
            // 2. Monitor canary metrics
            double successRate = monitorCanaryMetrics(canaryDeployment, request.getInitialTrafficPercentage());
            
            // 3. Gradually increase traffic based on metrics
            if (successRate >= request.getSuccessThreshold()) {
                return graduallyIncreaseTraffic(canaryDeployment, request);
            } else {
                // Rollback canary deployment
                rollbackCanaryDeployment(canaryDeployment);
                return new DeploymentResult(false, "Canary deployment failed metrics check");
            }
            
        } catch (Exception e) {
            logger.error("Canary deployment failed", e);
            throw new DeploymentException("Canary deployment failed: " + e.getMessage(), e);
        }
    }
    
    private String deployCanary(CanaryDeploymentRequest request) throws DeploymentException {
        try {
            // Create canary deployment with specified traffic percentage
            Deployment canary = new DeploymentBuilder()
                .withNewMetadata()
                    .withName("canary-" + request.getServiceName())
                    .withLabels(Collections.singletonMap("version", "canary"))
                .endMetadata()
                .withNewSpec()
                    .withReplicas(request.getInitialReplicas())
                    .withNewSelector()
                        .withMatchLabels(Collections.singletonMap("version", "canary"))
                    .endSelector()
                    .withNewTemplate()
                        .withNewMetadata()
                            .withLabels(Collections.singletonMap("version", "canary"))
                        .endMetadata()
                        .withNewSpec()
                            .addNewContainer()
                                .withName(request.getServiceName())
                                .withImage(request.getNewImage())
                                .addNewPort()
                                    .withContainerPort(8080)
                                .endPort()
                                .withResources(new ResourceRequirementsBuilder()
                                    .addToRequests("cpu", new Quantity("100m"))
                                    .addToRequests("memory", new Quantity("256Mi"))
                                    .addToLimits("cpu", new Quantity("200m"))
                                    .addToLimits("memory", new Quantity("512Mi"))
                                    .build())
                            .endContainer()
                        .endSpec()
                    .endTemplate()
                .endSpec()
                .build();
            
            k8sClient.apps().deployments().create(canary);
            logger.info("Canary deployment created: {}", canary.getMetadata().getName());
            
            // Configure traffic splitting (e.g., using Istio or similar service mesh)
            configureTrafficSplitting(request.getServiceName(), request.getInitialTrafficPercentage());
            
            return canary.getMetadata().getName();
            
        } catch (Exception e) {
            throw new DeploymentException("Failed to deploy canary: " + e.getMessage(), e);
        }
    }
    
    private double monitorCanaryMetrics(String deploymentName, int trafficPercentage) {
        // Monitor key metrics like error rate, response time, throughput
        double errorRate = metricsService.getErrorRate(deploymentName);
        double responseTime = metricsService.getAverageResponseTime(deploymentName);
        double throughput = metricsService.getThroughput(deploymentName);
        
        logger.info("Canary metrics - Error Rate: {}, Response Time: {}ms, Throughput: {} req/s", 
                   errorRate, responseTime, throughput);
        
        // Calculate success rate (simplified)
        return 1.0 - errorRate;
    }
    
    private DeploymentResult graduallyIncreaseTraffic(String deploymentName, 
                                                    CanaryDeploymentRequest request) {
        int currentTrafficPercentage = request.getInitialTrafficPercentage();
        int maxTrafficPercentage = request.getMaxTrafficPercentage();
        int increment = request.getTrafficIncrement();
        
        while (currentTrafficPercentage < maxTrafficPercentage) {
            try {
                // Wait for observation period
                Thread.sleep(request.getObservationPeriodMinutes() * 60 * 1000);
                
                // Monitor metrics
                double successRate = monitorCanaryMetrics(deploymentName, currentTrafficPercentage);
                
                if (successRate < request.getSuccessThreshold()) {
                    rollbackCanaryDeployment(deploymentName);
                    return new DeploymentResult(false, 
                        "Canary deployment failed metrics check at " + currentTrafficPercentage + "% traffic");
                }
                
                // Increase traffic
                currentTrafficPercentage = Math.min(
                    currentTrafficPercentage + increment, 
                    maxTrafficPercentage
                );
                
                updateTrafficSplitting(request.getServiceName(), currentTrafficPercentage);
                logger.info("Increased canary traffic to {}%", currentTrafficPercentage);
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                rollbackCanaryDeployment(deploymentName);
                return new DeploymentResult(false, "Canary deployment interrupted");
            } catch (Exception e) {
                logger.error("Error during traffic increase", e);
                rollbackCanaryDeployment(deploymentName);
                return new DeploymentResult(false, "Error during traffic increase: " + e.getMessage());
            }
        }
        
        // Promote canary to production
        return promoteCanaryToProduction(deploymentName, request);
    }
    
    private void configureTrafficSplitting(String serviceName, int percentage) {
        // Implementation for configuring traffic splitting
        // This would typically involve service mesh configuration (Istio, Linkerd, etc.)
        logger.info("Configuring traffic splitting for {} with {}% to canary", serviceName, percentage);
    }
    
    private void updateTrafficSplitting(String serviceName, int percentage) {
        // Implementation for updating traffic splitting
        logger.info("Updating traffic splitting for {} to {}% canary", serviceName, percentage);
    }
    
    private void rollbackCanaryDeployment(String deploymentName) {
        try {
            // Scale down canary deployment
            k8sClient.apps().deployments().withName(deploymentName).scale(0);
            
            // Remove canary deployment
            k8sClient.apps().deployments().withName(deploymentName).delete();
            
            // Reset traffic splitting
            resetTrafficSplitting(deploymentName);
            
            logger.info("Canary deployment rolled back: {}", deploymentName);
        } catch (Exception e) {
            logger.error("Error rolling back canary deployment: {}", deploymentName, e);
        }
    }
    
    private void resetTrafficSplitting(String deploymentName) {
        // Implementation for resetting traffic splitting
        logger.info("Resetting traffic splitting after rollback of {}", deploymentName);
    }
    
    private DeploymentResult promoteCanaryToProduction(String canaryDeploymentName, 
                                                     CanaryDeploymentRequest request) {
        try {
            // Scale up canary to full production capacity
            k8sClient.apps().deployments().withName(canaryDeploymentName)
                .scale(request.getProductionReplicas());
            
            // Update production deployment to use new image
            updateProductionDeployment(request);
            
            // Route 100% traffic to new version
            updateTrafficSplitting(request.getServiceName(), 100);
            
            // Clean up canary deployment
            k8sClient.apps().deployments().withName(canaryDeploymentName).delete();
            
            logger.info("Canary deployment promoted to production");
            return new DeploymentResult(true, "Canary deployment successfully promoted to production");
            
        } catch (Exception e) {
            logger.error("Error promoting canary to production", e);
            return new DeploymentResult(false, "Error promoting canary to production: " + e.getMessage());
        }
    }
    
    private void updateProductionDeployment(CanaryDeploymentRequest request) {
        // Implementation for updating production deployment
        logger.info("Updating production deployment with new image: {}", request.getNewImage());
    }
}
```

## Monitoring and Observability

Comprehensive monitoring provides visibility into system health, performance, and user experience.

### Application Metrics with Micrometer

```java
// MetricsConfiguration.java - Configuring application metrics
@Configuration
@EnableScheduling
public class MetricsConfiguration {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "myapp")
            .commonTags("environment", System.getenv("SPRING_PROFILES_ACTIVE"))
            .commonTags("version", "1.0.0");
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
    
    @Bean
    public CounterService counterService(MeterRegistry meterRegistry) {
        return new CounterService(meterRegistry);
    }
    
    @Bean
    public TimerService timerService(MeterRegistry meterRegistry) {
        return new TimerService(meterRegistry);
    }
}

// Custom metrics services
@Service
public class CounterService {
    
    private final Counter apiRequestCounter;
    private final Counter apiErrorCounter;
    private final Counter databaseQueryCounter;
    
    public CounterService(MeterRegistry meterRegistry) {
        this.apiRequestCounter = Counter.builder("api.requests.total")
            .description("Total API requests")
            .register(meterRegistry);
            
        this.apiErrorCounter = Counter.builder("api.errors.total")
            .description("Total API errors")
            .register(meterRegistry);
            
        this.databaseQueryCounter = Counter.builder("database.queries.total")
            .description("Total database queries")
            .register(meterRegistry);
    }
    
    public void incrementApiRequests() {
        apiRequestCounter.increment();
    }
    
    public void incrementApiErrors() {
        apiErrorCounter.increment();
    }
    
    public void incrementDatabaseQueries() {
        databaseQueryCounter.increment();
    }
}

@Service
public class TimerService {
    
    private final Timer apiResponseTimer;
    private final Timer databaseQueryTimer;
    private final Timer cacheOperationTimer;
    
    public TimerService(MeterRegistry meterRegistry) {
        this.apiResponseTimer = Timer.builder("api.response.time")
            .description("API response time")
            .register(meterRegistry);
            
        this.databaseQueryTimer = Timer.builder("database.query.time")
            .description("Database query execution time")
            .register(meterRegistry);
            
        this.cacheOperationTimer = Timer.builder("cache.operation.time")
            .description("Cache operation execution time")
            .register(meterRegistry);
    }
    
    public <T> T recordApiResponseTime(Supplier<T> operation) {
        return apiResponseTimer.record(operation);
    }
    
    public void recordDatabaseQuery(Runnable operation) {
        databaseQueryTimer.record(operation);
    }
    
    public <T> T recordCacheOperation(Supplier<T> operation) {
        return cacheOperationTimer.record(operation);
    }
}

// Business service with metrics
@Service
public class UserService {
    
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private CounterService counterService;
    
    @Autowired
    private TimerService timerService;
    
    @Timed(name = "user.service.find.by.id", description = "Time taken to find user by ID")
    public Optional<User> findById(Long id) {
        counterService.incrementApiRequests();
        
        return timerService.recordApiResponseTime(() -> {
            try {
                Optional<User> user = userRepository.findById(id);
                if (user.isEmpty()) {
                    counterService.incrementApiErrors();
                }
                return user;
            } catch (Exception e) {
                counterService.incrementApiErrors();
                logger.error("Error finding user by ID: {}", id, e);
                throw e;
            }
        });
    }
    
    @Timed(name = "user.service.create", description = "Time taken to create user")
    public User create(User user) {
        counterService.incrementApiRequests();
        
        return timerService.recordApiResponseTime(() -> {
            try {
                User savedUser = userRepository.save(user);
                logger.info("Created user with ID: {}", savedUser.getId());
                return savedUser;
            } catch (Exception e) {
                counterService.incrementApiErrors();
                logger.error("Error creating user: {}", user, e);
                throw e;
            }
        });
    }
    
    @Timed(name = "user.service.search", description = "Time taken to search users")
    public List<User> search(String query) {
        counterService.incrementApiRequests();
        
        return timerService.recordApiResponseTime(() -> {
            try {
                List<User> users = userRepository.findByUsernameContaining(query);
                logger.info("Found {} users for query: {}", users.size(), query);
                return users;
            } catch (Exception e) {
                counterService.incrementApiErrors();
                logger.error("Error searching users for query: {}", query, e);
                throw e;
            }
        });
    }
}
```

### Distributed Tracing with OpenTelemetry

```java
// TracingConfiguration.java - Configuring distributed tracing
@Configuration
public class TracingConfiguration {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(SdkTracerProvider.builder()
                .addSpanProcessor(BatchSpanProcessor.builder(
                    OtlpGrpcSpanExporter.builder()
                        .setEndpoint("http://localhost:4317")
                        .build())
                    .build())
                .setResource(Resource.getDefault()
                    .toBuilder()
                    .put(ResourceAttributes.SERVICE_NAME, "myapp")
                    .put(ResourceAttributes.SERVICE_VERSION, "1.0.0")
                    .build())
                .build())
            .setPropagators(ContextPropagators.create(
                TextMapPropagator.composite(
                    W3CTraceContextPropagator.getInstance(),
                    W3CBaggagePropagator.getInstance())))
            .buildAndRegisterGlobal();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("myapp");
    }
}

// Traced service with manual instrumentation
@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    public Order processOrder(OrderRequest request) {
        Span span = tracer.spanBuilder("processOrder")
            .setAttribute("order.id", request.getOrderId())
            .setAttribute("order.amount", request.getAmount())
            .startSpan();
            
        try (Scope scope = span.makeCurrent()) {
            logger.info("Processing order: {}", request.getOrderId());
            
            // Check inventory
            Span inventorySpan = tracer.spanBuilder("checkInventory")
                .setAttribute("product.id", request.getProductId())
                .startSpan();
                
            try (Scope inventoryScope = inventorySpan.makeCurrent()) {
                boolean inStock = inventoryService.checkStock(request.getProductId(), request.getQuantity());
                inventorySpan.setAttribute("product.inStock", inStock);
                
                if (!inStock) {
                    span.setStatus(StatusCode.ERROR, "Product out of stock");
                    throw new InsufficientStockException("Product not available");
                }
            } finally {
                inventorySpan.end();
            }
            
            // Process payment
            Span paymentSpan = tracer.spanBuilder("processPayment")
                .setAttribute("payment.amount", request.getAmount())
                .startSpan();
                
            try (Scope paymentScope = paymentSpan.makeCurrent()) {
                PaymentResult paymentResult = paymentService.charge(request.getPaymentInfo(), request.getAmount());
                paymentSpan.setAttribute("payment.success", paymentResult.isSuccess());
                paymentSpan.setAttribute("payment.transactionId", paymentResult.getTransactionId());
                
                if (!paymentResult.isSuccess()) {
                    span.setStatus(StatusCode.ERROR, "Payment failed");
                    throw new PaymentException("Payment processing failed");
                }
            } finally {
                paymentSpan.end();
            }
            
            // Create order
            Order order = createOrder(request);
            span.setAttribute("order.created", true);
            span.setAttribute("order.status", order.getStatus().toString());
            
            logger.info("Order processed successfully: {}", order.getId());
            return order;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            logger.error("Error processing order: {}", request.getOrderId(), e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private Order createOrder(OrderRequest request) {
        // Implementation for creating order
        Order order = new Order();
        order.setId(request.getOrderId());
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setAmount(request.getAmount());
        order.setStatus(OrderStatus.CONFIRMED);
        order.setCreatedAt(LocalDateTime.now());
        return order;
    }
}

// HTTP client with tracing
@Component
public class ExternalPaymentClient {
    
    @Autowired
    private Tracer tracer;
    
    private final HttpClient httpClient = HttpClient.newHttpClient();
    
    public PaymentResponse processPayment(PaymentRequest request) throws IOException, InterruptedException {
        Span span = tracer.spanBuilder("externalPaymentCall")
            .setAttribute("payment.provider", "external")
            .startSpan();
            
        try (Scope scope = span.makeCurrent()) {
            String requestBody = objectMapper.writeValueAsString(request);
            
            HttpRequest httpRequest = HttpRequest.newBuilder()
                .uri(URI.create("https://payment-api.example.com/charge"))
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer " + getApiKey())
                .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                .build();
            
            // Inject trace context into HTTP headers
            TextMapPropagator propagator = OpenTelemetry.getGlobalPropagators().getTextMapPropagator();
            propagator.inject(Context.current(), httpRequest, (carrier, key, value) -> {
                if (carrier instanceof HttpRequest.Builder) {
                    ((HttpRequest.Builder) carrier).header(key, value);
                }
            });
            
            HttpResponse<String> response = httpClient.send(httpRequest, HttpResponse.BodyHandlers.ofString());
            
            span.setAttribute("http.status_code", response.statusCode());
            
            if (response.statusCode() == 200) {
                return objectMapper.readValue(response.body(), PaymentResponse.class);
            } else {
                span.setStatus(StatusCode.ERROR, "HTTP " + response.statusCode());
                throw new PaymentException("Payment API returned status: " + response.statusCode());
            }
            
        } finally {
            span.end();
        }
    }
}
```

## Security and Compliance Automation

Automated security scanning and compliance checks are essential for maintaining secure deployments.

### Security Scanning in Pipelines

```groovy
// Jenkins pipeline with security scanning
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.1'
        jdk 'OpenJDK 17'
    }
    
    stages {
        stage('Security Analysis') {
            parallel {
                stage('Dependency Scanning') {
                    steps {
                        sh 'mvn dependency-check:check'
                    }
                    post {
                        always {
                            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                        }
                    }
                }
                
                stage('Static Analysis') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                    post {
                        always {
                            recordIssues tools: [spotBugs(pattern: 'target/spotbugsXml.xml')]
                        }
                    }
                }
                
                stage('Container Scanning') {
                    steps {
                        script {
                            def imageTag = "myapp:${env.BUILD_NUMBER}"
                            sh "docker build -t ${imageTag} ."
                            sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${imageTag}"
                        }
                    }
                }
            }
        }
        
        stage('Compliance Check') {
            steps {
                sh 'checkov -d . --quiet'
            }
        }
    }
}
```

### Security Configuration

```java
// SecurityConfiguration.java - Spring Security with comprehensive protection
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfiguration {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .requireCsrfProtectionMatcher(new CustomCsrfMatcher()))
            .headers(headers -> headers
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .contentTypeOptions(ContentTypeOptionsConfig::disable)
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                    .preload(true)))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .maximumSessions(1)
                .maxSessionsPreventsLogin(true))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())))
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
                .accessDeniedHandler(new BearerTokenAccessDeniedHandler()));
            
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder jwtDecoder = NimbusJwtDecoder.withJwkSetUri(jwkSetUri)
            .build();
            
        // Add custom validator for additional claims
        OAuth2TokenValidator<Jwt> validator = new DelegatingOAuth2TokenValidator<>(
            new JwtTimestampValidator(),
            new JwtIssuerValidator(issuerUri),
            new CustomClaimValidator()
        );
        
        jwtDecoder.setJwtValidator(validator);
        return jwtDecoder;
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    // Custom CSRF matcher to exclude API endpoints
    private static class CustomCsrfMatcher implements RequestMatcher {
        @Override
        public boolean matches(HttpServletRequest request) {
            return !request.getRequestURI().startsWith("/api/");
        }
    }
}

// Custom claim validator for JWT tokens
@Component
public class CustomClaimValidator implements OAuth2TokenValidator<Jwt> {
    
    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        // Validate custom claims like tenant, permissions, etc.
        String tenant = jwt.getClaimAsString("tenant");
        List<String> permissions = jwt.getClaimAsStringList("permissions");
        
        if (tenant == null || tenant.isEmpty()) {
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "Tenant claim is missing", null));
        }
        
        if (permissions == null || permissions.isEmpty()) {
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "Permissions claim is missing", null));
        }
        
        return OAuth2TokenValidatorResult.success();
    }
}

// Security audit logging
@Component
public class SecurityAuditLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(SecurityAuditLogger.class);
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getPrincipal().toString();
        logger.info("Authentication successful for user: {}", username);
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getPrincipal().toString();
        String errorMessage = event.getException().getMessage();
        logger.warn("Authentication failed for user: {} - Error: {}", username, errorMessage);
    }
    
    @EventListener
    public void handleAuthorizationFailure(AuthorizationDeniedEvent event) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String username = authentication != null ? authentication.getPrincipal().toString() : "anonymous";
        String requestURI = ((FilterInvocation) event.getSource()).getRequest().getRequestURI();
        logger.warn("Authorization denied for user: {} on resource: {}", username, requestURI);
    }
}
```

## Performance Optimization

Performance optimization ensures efficient resource utilization and responsive user experiences.

### JVM Tuning for CI/CD Environments

```bash
#!/bin/bash
# jvm-tuning.sh - JVM optimization parameters for CI/CD environments

# Memory settings
JVM_OPTS="$JVM_OPTS -Xms2g"
JVM_OPTS="$JVM_OPTS -Xmx4g"
JVM_OPTS="$JVM_OPTS -XX:NewRatio=1"
JVM_OPTS="$JVM_OPTS -XX:SurvivorRatio=8"
JVM_OPTS="$JVM_OPTS -XX:MaxMetaspaceSize=512m"

# Garbage collection tuning
JVM_OPTS="$JVM_OPTS -XX:+UseG1GC"
JVM_OPTS="$JVM_OPTS -XX:MaxGCPauseMillis=200"
JVM_OPTS="$JVM_OPTS -XX:G1HeapRegionSize=16m"
JVM_OPTS="$JVM_OPTS -XX:+G1UseAdaptiveIHOP"
JVM_OPTS="$JVM_OPTS -XX:G1MixedGCCountTarget=8"

# JIT compilation
JVM_OPTS="$JVM_OPTS -XX:CompileThreshold=1000"
JVM_OPTS="$JVM_OPTS -XX:TieredStopAtLevel=1"

# Monitoring and debugging
JVM_OPTS="$JVM_OPTS -XX:+PrintGC"
JVM_OPTS="$JVM_OPTS -XX:+PrintGCDetails"
JVM_OPTS="$JVM_OPTS -XX:+PrintGCTimeStamps"
JVM_OPTS="$JVM_OPTS -Xloggc:gc.log"
JVM_OPTS="$JVM_OPTS -XX:+UseGCLogFileRotation"
JVM_OPTS="$JVM_OPTS -XX:NumberOfGCLogFiles=5"
JVM_OPTS="$JVM_OPTS -XX:GCLogFileSize=100M"

# Flight recorder for performance analysis
JVM_OPTS="$JVM_OPTS -XX:StartFlightRecording=duration=300s,filename=app-profile.jfr,settings=profile"

echo "JVM_OPTS: $JVM_OPTS"
```

### Pipeline Performance Optimization

```yaml
# .gitlab-ci.yml with performance optimizations
stages:
  - prepare
  - build
  - test
  - deploy

variables:
  MAVEN_OPTS: >
    -Dmaven.repo.local=.m2/repository
    -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN
  DOCKER_DRIVER: overlay2
  GRADLE_OPTS: "-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true -Dorg.gradle.workers.max=2"

# Cache configuration for faster builds
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .m2/repository/
    - .gradle/
    - node_modules/

# Parallel build jobs
build_parallel:
  stage: build
  image: maven:3.8-openjdk-17
  script:
    - mvn $MAVEN_OPTS compile -T 1C  # Use all available cores
  artifacts:
    paths:
      - target/classes/
    expire_in: 1 day
  cache:
    key: maven-$CI_COMMIT_REF_SLUG
    paths:
      - .m2/repository/

# Matrix testing for multiple configurations
test_matrix:
  stage: test
  image: maven:3.8-openjdk-17
  parallel:
    matrix:
      - JAVA_VERSION: ["17", "21"]
        DATABASE: ["mysql", "postgresql"]
  script:
    - echo "Testing with Java $JAVA_VERSION and $DATABASE"
    - mvn $MAVEN_OPTS test -Dspring.profiles.active=test-$DATABASE
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

# Optimized deployment
deploy_optimized:
  stage: deploy
  image: google/cloud-sdk:alpine
  before_script:
    - echo $GCLOUD_SERVICE_KEY | gcloud auth activate-service-account --key-file=-
    - gcloud --quiet config set project $GCP_PROJECT_ID
    - gcloud --quiet config set compute/zone $GCP_ZONE
  script:
    - |
      # Deploy with rolling update strategy
      gcloud run deploy myapp \
        --image gcr.io/$GCP_PROJECT_ID/myapp:$CI_COMMIT_SHA \
        --platform managed \
        --region $GCP_REGION \
        --allow-unauthenticated \
        --memory 1Gi \
        --cpu 1 \
        --concurrency 80 \
        --max-instances 100 \
        --set-env-vars=SPRING_PROFILES_ACTIVE=production
  environment:
    name: production
    url: https://myapp.run.app
```

## Best Practices and Anti-Patterns

Understanding best practices and common anti-patterns is crucial for successful DevOps implementation.

### Best Practices

1. **Infrastructure as Code**: Treat infrastructure like application code with version control, testing, and peer review
2. **Immutable Infrastructure**: Create new instances instead of modifying existing ones
3. **Automated Testing**: Implement comprehensive test suites that run automatically in pipelines
4. **Monitoring and Alerting**: Establish proactive monitoring with meaningful alerts
5. **Security by Design**: Integrate security checks throughout the pipeline
6. **Documentation**: Maintain up-to-date documentation for processes and configurations

### Common Anti-Patterns

1. **Snowflake Servers**: Manually configured servers that can't be easily reproduced
2. **Manual Deployments**: Relying on manual steps instead of automation
3. **Long-Lived Branches**: Maintaining feature branches for extended periods
4. **Insufficient Testing**: Skipping or minimizing automated testing
5. **Poor Error Handling**: Not implementing proper error recovery mechanisms
6. **Inadequate Monitoring**: Deploying without proper observability

### Pipeline Optimization Techniques

```java
// PipelineOptimizationService.java - Service for pipeline optimization
@Service
public class PipelineOptimizationService {
    
    private static final Logger logger = LoggerFactory.getLogger(PipelineOptimizationService.class);
    
    @Autowired
    private BuildMetricsRepository metricsRepository;
    
    public PipelineOptimizationReport analyzePipelinePerformance() {
        PipelineOptimizationReport report = new PipelineOptimizationReport();
        
        // Analyze build times
        List<BuildMetric> recentBuilds = metricsRepository.findRecentBuilds(100);
        BuildTimeAnalysis buildTimeAnalysis = analyzeBuildTimes(recentBuilds);
        report.setBuildTimeAnalysis(buildTimeAnalysis);
        
        // Analyze test performance
        TestPerformanceAnalysis testAnalysis = analyzeTestPerformance(recentBuilds);
        report.setTestPerformanceAnalysis(testAnalysis);
        
        // Identify bottlenecks
        List<Bottleneck> bottlenecks = identifyBottlenecks(recentBuilds);
        report.setBottlenecks(bottlenecks);
        
        // Generate recommendations
        List<Recommendation> recommendations = generateRecommendations(report);
        report.setRecommendations(recommendations);
        
        return report;
    }
    
    private BuildTimeAnalysis analyzeBuildTimes(List<BuildMetric> builds) {
        BuildTimeAnalysis analysis = new BuildTimeAnalysis();
        
        // Calculate statistics
        DoubleSummaryStatistics stats = builds.stream()
            .mapToDouble(BuildMetric::getDurationSeconds)
            .summaryStatistics();
            
        analysis.setAverageDuration(stats.getAverage());
        analysis.setMinDuration(stats.getMin());
        analysis.setMaxDuration(stats.getMax());
        analysis.setTotalBuilds(stats.getCount());
        
        // Identify slow builds
        double threshold = stats.getAverage() * 1.5; // 50% above average
        List<BuildMetric> slowBuilds = builds.stream()
            .filter(build -> build.getDurationSeconds() > threshold)
            .collect(Collectors.toList());
            
        analysis.setSlowBuilds(slowBuilds);
        
        return analysis;
    }
    
    private TestPerformanceAnalysis analyzeTestPerformance(List<BuildMetric> builds) {
        TestPerformanceAnalysis analysis = new TestPerformanceAnalysis();
        
        // Analyze test execution times
        DoubleSummaryStatistics testStats = builds.stream()
            .mapToDouble(BuildMetric::getTestDurationSeconds)
            .summaryStatistics();
            
        analysis.setAverageTestDuration(testStats.getAverage());
        analysis.setTotalTestExecutions(testStats.getCount());
        
        // Identify flaky tests
        Map<String, Long> testFailureCounts = builds.stream()
            .flatMap(build -> build.getFailedTests().stream())
            .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
            
        List<String> flakyTests = testFailureCounts.entrySet().stream()
            .filter(entry -> entry.getValue() > 1 && entry.getValue() < 10) // Failed 2-9 times
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
            
        analysis.setFlakyTests(flakyTests);
        
        return analysis;
    }
    
    private List<Bottleneck> identifyBottlenecks(List<BuildMetric> builds) {
        List<Bottleneck> bottlenecks = new ArrayList<>();
        
        // Analyze stage durations
        Map<String, DoubleSummaryStatistics> stageStats = builds.stream()
            .flatMap(build -> build.getStageDurations().entrySet().stream())
            .collect(Collectors.groupingBy(
                Map.Entry::getKey,
                Collectors.summarizingDouble(Map.Entry::getValue)
            ));
        
        // Identify stages with high variance or long average duration
        stageStats.forEach((stageName, stats) -> {
            double average = stats.getAverage();
            double max = stats.getMax();
            
            if (average > 300) { // More than 5 minutes on average
                bottlenecks.add(new Bottleneck(
                    "Stage: " + stageName,
                    "Average duration: " + average + " seconds",
                    BottleneckSeverity.HIGH
                ));
            }
            
            if ((max - average) > 600) { // High variance (more than 10 minutes difference)
                bottlenecks.add(new Bottleneck(
                    "Stage: " + stageName,
                    "High variance in execution time",
                    BottleneckSeverity.MEDIUM
                ));
            }
        });
        
        return bottlenecks;
    }
    
    private List<Recommendation> generateRecommendations(PipelineOptimizationReport report) {
        List<Recommendation> recommendations = new ArrayList<>();
        
        // Recommendations based on build time analysis
        if (report.getBuildTimeAnalysis().getAverageDuration() > 1800) { // More than 30 minutes
            recommendations.add(new Recommendation(
                "Optimize build process",
                "Consider parallelizing build steps or optimizing dependencies",
                RecommendationPriority.HIGH
            ));
        }
        
        // Recommendations based on test performance
        if (!report.getTestPerformanceAnalysis().getFlakyTests().isEmpty()) {
            recommendations.add(new Recommendation(
                "Fix flaky tests",
                "Address flaky tests to improve pipeline reliability: " + 
                String.join(", ", report.getTestPerformanceAnalysis().getFlakyTests()),
                RecommendationPriority.HIGH
            ));
        }
        
        // Recommendations based on bottlenecks
        for (Bottleneck bottleneck : report.getBottlenecks()) {
            recommendations.add(new Recommendation(
                "Address pipeline bottleneck",
                bottleneck.getDescription(),
                RecommendationPriority.MEDIUM
            ));
        }
        
        return recommendations;
    }
}
```

## Conclusion

Advanced DevOps and CI/CD pipelines are essential for modern software delivery, providing automation, reliability, and scalability. Key takeaways include:

1. **Comprehensive Pipeline Architecture**: Multi-stage pipelines with automated testing, security scanning, and deployment
2. **Infrastructure as Code**: Version-controlled infrastructure provisioning with tools like Terraform
3. **Containerization and Orchestration**: Docker containers with Kubernetes orchestration for consistent deployments
4. **Advanced Deployment Strategies**: Blue-green and canary deployments for risk reduction
5. **Monitoring and Observability**: Comprehensive metrics, logging, and distributed tracing
6. **Security Automation**: Integrated security scanning and compliance checks
7. **Performance Optimization**: JVM tuning and pipeline optimization techniques
8. **Best Practices**: Following established patterns while avoiding common anti-patterns

By implementing these advanced DevOps practices, organizations can achieve faster delivery cycles, improved reliability, and enhanced security while maintaining high-quality software products.