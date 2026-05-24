# Module 08: Kubernetes + Cloud Architecture — Progress Tracker

**Start Date:** _______________ | **Target:** 3 weeks | **Actual Completion:** _______________

---

## Week 1: Docker + Kubernetes Fundamentals

| Task | Done |
|------|------|
| Write a Dockerfile for your payment-ai-service (use multi-stage build) | ☐ |
| Build and run the image locally: `docker build` + `docker run` | ☐ |
| Set up a local Kubernetes cluster (Minikube or Kind) | ☐ |
| Write `deployment.yaml` with 3 replicas, resource limits, liveness/readiness probes | ☐ |
| Write `service.yaml` (ClusterIP) and expose via `kubectl port-forward` | ☐ |
| Create a Kubernetes Secret for OPENAI_API_KEY — mount as env variable | ☐ |
| Deploy Qdrant as a StatefulSet with a PersistentVolumeClaim | ☐ |
| Research Q1, Q2 answered | ☐ |

---

## Week 2: Autoscaling + CI/CD

| Task | Done |
|------|------|
| Configure HPA: scale on CPU (70% threshold), min=3, max=10 | ☐ |
| Load test with k6 or Apache JMeter: observe HPA scaling up pods | ☐ |
| Write a GitHub Actions workflow: test → build Docker image → push to registry | ☐ |
| Add kubectl deploy step: `kubectl set image` + `kubectl rollout status` | ☐ |
| Test rolling deployment: deploy new version → verify no downtime | ☐ |
| Test rollback: `kubectl rollout undo` after a bad deployment | ☐ |
| Configure graceful shutdown: `server.shutdown: graceful` + verify in-flight requests complete | ☐ |
| Research Q3, Q4, Q5 answered | ☐ |

---

## Week 3: Terraform + Cloud

| Task | Done |
|------|------|
| Install Terraform locally and configure AWS credentials | ☐ |
| Write Terraform for a VPC with public/private subnets | ☐ |
| Write Terraform for an RDS PostgreSQL instance with Multi-AZ | ☐ |
| Run `terraform plan` and review the resource plan before applying | ☐ |
| Write Helm chart for payment-ai-service (Chart.yaml + values.yaml + templates/) | ☐ |
| Deploy with Helm: `helm install` and `helm upgrade` | ☐ |
| Write a production readiness checklist for your AI service | ☐ |
| Research Q6, Q7, Q8, Q9, Q10 answered | ☐ |

---

**Final Gate:** Can you write a Dockerfile, Kubernetes deployment YAML, and a basic Terraform file? Can you explain resource limits, secrets management, and HPA? YES / NO

---

## Skills Self-Assessment

| Skill | Rating /5 |
|-------|-----------|
| Writing production Dockerfiles | |
| Kubernetes deployment YAML (resources, probes, secrets) | |
| Kubernetes StatefulSet (for databases) | |
| HPA configuration (CPU + custom metrics) | ☐ |
| CI/CD pipeline with GitHub Actions | |
| Graceful shutdown in Spring Boot + Kubernetes | |
| Terraform (VPC, RDS, EKS basics) | |
| Helm chart structure and usage | |
| Kubernetes network security (NetworkPolicy) | |
| Cloud cost optimization strategies | |

---

## Time Log

| Week | Hours | Key Insight |
|------|-------|-------------|
| Week 1 — Docker + K8s Fundamentals | | |
| Week 2 — Autoscaling + CI/CD | | |
| Week 3 — Terraform + Cloud | | |
| **Total** | | |
