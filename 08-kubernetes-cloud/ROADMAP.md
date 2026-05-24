# Module 08: Kubernetes + Cloud Architecture
**Duration:** 3 weeks (2 hours/day)
**Goal:** Deploy Spring Boot and AI services to Kubernetes, manage cloud infrastructure with Terraform, and understand cloud-native patterns.

---

## The Big Idea

```
Writing good code is not enough.
Architects must know how to DEPLOY and OPERATE their systems.

Kubernetes (K8s): runs your containers reliably at scale
  - Automatic restarts if your service crashes
  - Scale horizontally when load increases
  - Rolling updates without downtime
  - Resource limits (prevent one service from eating all CPU)

Cloud (AWS/GCP/Azure): infrastructure you don't manage
  - Managed PostgreSQL (RDS): no DBA needed
  - Managed Kafka (MSK / Confluent Cloud): no Kafka ops needed
  - LLM APIs (Azure OpenAI): no GPU management
  - Object storage (S3): store documents for RAG ingestion

Terraform: define infrastructure as code
  - Reproducible: same Terraform = same infrastructure, every time
  - Version controlled: infrastructure changes are in Git
  - Review-able: infrastructure changes go through code review

For AI systems specifically:
  - Ollama runs in K8s (needs GPU nodes for production)
  - Vector DB (Qdrant) deployed as a StatefulSet
  - LLM inference can be autoscaled based on queue depth
```

---

## Week 1: Docker + Kubernetes Fundamentals

**Monday — Docker for Spring Boot AI Apps**
```dockerfile
# Dockerfile for Spring Boot AI service
FROM eclipse-temurin:21-jre-jammy AS runtime

WORKDIR /app

# Add non-root user (security best practice)
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

COPY target/payment-ai-service.jar app.jar

# JVM optimizations for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75 \
               -XX:+UseG1GC \
               -Djava.security.egd=file:/dev/./urandom"

USER appuser

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]

# Multi-stage build for smaller images:
# FROM maven:3.9-eclipse-temurin-21 AS build
# COPY pom.xml .
# COPY src ./src
# RUN mvn package -DskipTests
# FROM eclipse-temurin:21-jre-jammy AS runtime
# COPY --from=build /app/target/*.jar app.jar
```

**Tuesday–Wednesday — Kubernetes Deployment**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-ai-service
  namespace: fintech
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-ai-service
  template:
    metadata:
      labels:
        app: payment-ai-service
    spec:
      containers:
        - name: payment-ai-service
          image: myregistry/payment-ai-service:1.2.0
          ports:
            - containerPort: 8080
          
          # Resource limits: CRITICAL — prevents one pod eating all resources
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          
          # Environment variables from secrets
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-secrets
                  key: openai-api-key
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-ai-service
  namespace: fintech
spec:
  selector:
    app: payment-ai-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ai-ingress
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"  # rate limit at ingress level
spec:
  rules:
    - host: api.paymentco.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: payment-ai-service
                port:
                  number: 80
```

**Thursday — Secrets Management**
```yaml
# Never store API keys in deployment YAML or code
# Use Kubernetes Secrets (or better: AWS Secrets Manager / Vault)

# Create secret (base64 encoded):
# kubectl create secret generic ai-secrets \
#   --from-literal=openai-api-key=sk-... \
#   --namespace=fintech

# For production: use External Secrets Operator (sync from AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-secrets
  namespace: fintech
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: ai-secrets
  data:
    - secretKey: openai-api-key
      remoteRef:
        key: /fintech/prod/openai-api-key
```

**Friday — Qdrant and Ollama on Kubernetes**
```yaml
# Qdrant StatefulSet: vector database needs persistent storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: fintech
spec:
  replicas: 1
  serviceName: qdrant
  selector:
    matchLabels:
      app: qdrant
  template:
    spec:
      containers:
        - name: qdrant
          image: qdrant/qdrant:latest
          ports:
            - containerPort: 6333
            - containerPort: 6334
          volumeMounts:
            - name: qdrant-storage
              mountPath: /qdrant/storage
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
  volumeClaimTemplates:
    - metadata:
        name: qdrant-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi

---
# Ollama: local LLM inference (needs GPU for production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: fintech
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
          volumeMounts:
            - name: ollama-models
              mountPath: /root/.ollama
          resources:
            limits:
              nvidia.com/gpu: "1"   # request 1 GPU (if using GPU nodes)
          env:
            - name: OLLAMA_MODELS
              value: "/root/.ollama/models"
```

---

## Week 2: Horizontal Pod Autoscaling + CI/CD

**Monday–Tuesday — Autoscaling**
```yaml
# HPA: scale based on CPU, memory, or custom metrics (like Kafka lag)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-ai-service-hpa
  namespace: fintech
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-ai-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    # Scale on CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Scale on Kafka consumer lag (KEDA custom metric)
    - type: External
      external:
        metric:
          name: kafka_consumer_group_lag
          selector:
            matchLabels:
              consumerGroup: "fraud-detection-group"
              topic: "payment-events"
        target:
          type: AverageValue
          averageValue: "100"  # scale up when lag > 100 events per replica

# KEDA (Kubernetes Event-Driven Autoscaling):
# Scale to zero when no Kafka messages (save cost)
# Scale up fast when messages arrive
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: fraud-consumer-scaler
spec:
  scaleTargetRef:
    name: fraud-detection-consumer
  minReplicaCount: 0     # scale to ZERO when idle
  maxReplicaCount: 50
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: fraud-detection-group
        topic: payment-events
        lagThreshold: "50"  # 1 replica per 50 messages in lag
```

**Wednesday–Thursday — CI/CD Pipeline**
```yaml
# GitHub Actions: build → test → push image → deploy to K8s
name: CI/CD

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run tests
        run: mvn test
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY_TEST }}

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: |
          mvn package -DskipTests
          docker build -t ${{ secrets.REGISTRY }}/payment-ai-service:${{ github.sha }} .
          docker push ${{ secrets.REGISTRY }}/payment-ai-service:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/payment-ai-service \
            payment-ai-service=${{ secrets.REGISTRY }}/payment-ai-service:${{ github.sha }} \
            --namespace=fintech
          kubectl rollout status deployment/payment-ai-service \
            --namespace=fintech \
            --timeout=5m
```

**Friday — Helm Charts**
```yaml
# Helm: package manager for Kubernetes
# Chart structure:
# payment-ai-service/
#   Chart.yaml       -- chart metadata
#   values.yaml      -- default configuration
#   templates/       -- Kubernetes YAML templates

# values.yaml:
replicaCount: 3
image:
  repository: myregistry/payment-ai-service
  tag: "1.2.0"
resources:
  limits:
    memory: "1Gi"
    cpu: "1000m"
openai:
  model: gpt-4o
  maxTokens: 2000
qdrant:
  host: qdrant
  port: 6334

# templates/deployment.yaml (uses values):
# image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
# replicas: {{ .Values.replicaCount }}

# Deploy with Helm:
# helm install payment-ai ./payment-ai-service -n fintech -f values-prod.yaml
# helm upgrade payment-ai ./payment-ai-service -n fintech --set image.tag=1.2.1
# helm rollback payment-ai 1  (rollback to previous version)
```

---

## Week 3: Terraform + Cloud Architecture

**Monday–Tuesday — Infrastructure as Code**
```hcl
# main.tf — AWS infrastructure for AI payment system

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "fintech-terraform-state"
    key    = "payment-ai/terraform.tfstate"
    region = "eu-west-2"
  }
}

# VPC (private network)
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = "payment-ai-vpc"
  cidr   = "10.0.0.0/16"
  azs    = ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
}

# EKS (managed Kubernetes)
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "payment-ai-cluster"
  cluster_version = "1.31"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  node_groups = {
    main = {
      desired_size = 3
      min_size     = 3
      max_size     = 20
      instance_types = ["t3.xlarge"]  # 4 vCPU, 16GB RAM
    }
  }
}

# RDS PostgreSQL with pgvector
resource "aws_db_instance" "main" {
  identifier           = "payment-ai-postgres"
  engine               = "postgres"
  engine_version       = "16.3"
  instance_class       = "db.t3.large"
  allocated_storage    = 100
  storage_encrypted    = true
  multi_az             = true   # high availability
  deletion_protection  = true   # prevent accidental deletion
  
  db_name  = "payments"
  username = "postgres"
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
}

# MSK (managed Kafka)
resource "aws_msk_cluster" "main" {
  cluster_name           = "payment-kafka"
  kafka_version          = "3.7.0"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = module.vpc.private_subnets
    storage_info {
      ebs_storage_info { volume_size = 1000 }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
  }
}
```

**Wednesday — Cost Optimization**
```
AWS cost optimization for AI systems:

1. Right-size EC2/EKS nodes:
   - Profile actual CPU/memory usage with kubectl top
   - Don't use t3.xlarge if t3.large is enough
   - Spot instances for stateless services (60-80% cheaper)

2. Scale to zero with KEDA:
   - Fraud analysis consumer: 0 replicas when no Kafka messages
   - Saves ~70% on idle node costs

3. Model cost optimization:
   - Use GPT-3.5 for simple tasks (classification, extraction)
   - Use GPT-4o only for complex generation
   - Semantic cache: 40% fewer API calls

4. Reserved instances:
   - For baseline load (3 always-on nodes): 1-year reserved = 40% savings
   - For burst capacity: on-demand or spot

5. S3 for documents:
   - Store PDFs in S3 (cheap: $0.023/GB)
   - Only keep embeddings in Qdrant/pgvector (expensive: memory)
   - Ingest from S3 on demand

Rough cost estimate (AWS, London region):
  3 × t3.xlarge EKS nodes:    £300/month
  db.t3.large RDS (Multi-AZ): £200/month
  MSK 3-node cluster:         £400/month
  OpenAI GPT-4o (100K calls): £500/month
  Total baseline:             ~£1,400/month
```

**Thursday–Friday — Production Checklist**
```
Before deploying an AI system to production:

INFRASTRUCTURE:
  ✅ Multi-AZ deployment (database + Kafka)
  ✅ Resource limits on all pods (prevents memory leaks killing the cluster)
  ✅ HPA configured (scale up under load)
  ✅ PodDisruptionBudget (maintain min replicas during node maintenance)
  ✅ Secrets in AWS Secrets Manager (not in YAML or environment variables)
  ✅ VPC with private subnets (DB and Kafka not internet-accessible)

AI SERVICES:
  ✅ Circuit breaker on LLM API calls (Resilience4j)
  ✅ Rate limiting on AI endpoints
  ✅ Token usage logging (cost visibility)
  ✅ Semantic cache (cost reduction)
  ✅ Fallback to local Ollama when cloud LLM is down
  ✅ Output sanitization (no card numbers in AI responses)

OBSERVABILITY:
  ✅ Prometheus metrics (CPU, memory, custom AI metrics)
  ✅ Grafana dashboards (latency, cost, error rate)
  ✅ Structured logging to CloudWatch/ELK
  ✅ Distributed tracing (Jaeger/Zipkin)
  ✅ Alerting: high error rate, high latency, high cost

SECURITY:
  ✅ TLS everywhere (in-transit encryption)
  ✅ Network policies (pod-to-pod communication restricted)
  ✅ RBAC (pods have minimum required permissions)
  ✅ Image scanning (Trivy in CI pipeline)
  ✅ No privileged containers

OPERATIONS:
  ✅ Runbook: how to restart services, scale up, rollback
  ✅ On-call rotation with escalation
  ✅ DR plan (how to recover from AZ failure)
  ✅ Data backup and restore tested
```
