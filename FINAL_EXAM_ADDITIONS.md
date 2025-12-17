
---

## 13. Infrastructure Provisioning (Terraform)

### Overview
Infrastructure is provisioned using Terraform to automate AWS resource creation following Infrastructure as Code (IaC) principles.

### AWS Resources Provisioned

| Resource | Description | Terraform File |
|----------|-------------|----------------|
| VPC | Virtual Private Cloud with 10.0.0.0/16 CIDR | `infra/main.tf` |
| Subnets | 2 public + 2 private subnets across AZs | `infra/main.tf` |
| Internet Gateway | For public internet access | `infra/main.tf` |
| NAT Gateway | For private subnet outbound access | `infra/main.tf` |
| Security Groups | App, DB, and EKS cluster SGs | `infra/main.tf` |
| EKS Cluster | Managed Kubernetes cluster | `infra/eks.tf` |
| EKS Node Group | Worker nodes (t3.medium) | `infra/eks.tf` |
| RDS PostgreSQL | Managed database (db.t3.micro) | `infra/rds.tf` |
| S3 Bucket | Application assets storage | `infra/rds.tf` |

### Terraform Commands

```bash
# Initialize Terraform
cd infra/
terraform init

# Plan infrastructure changes
terraform plan -out=tfplan

# Apply infrastructure
terraform apply tfplan

# View outputs
terraform output

# Destroy infrastructure (cleanup)
terraform destroy
```

### Security Considerations
- No hardcoded AWS credentials (use environment variables or IAM roles)
- Database password stored in `terraform.tfvars` (gitignored)
- S3 bucket with encryption and public access blocked
- Security groups follow principle of least privilege

---

## 14. Kubernetes Deployment

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    EKS Cluster / Minikube                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Ingress   │──│  Frontend   │──│   Backend   │          │
│  │   Controller│  │  (2 pods)   │  │  (2 pods)   │          │
│  └─────────────┘  └─────────────┘  └──────┬──────┘          │
│                                           │                  │
│  ┌─────────────┐  ┌─────────────┐         │                  │
│  │    Redis    │  │  PostgreSQL │◄────────┘                  │
│  │   (cache)   │  │   (1 pod)   │                            │
│  └─────────────┘  └─────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### Kubernetes Manifests

| File | Purpose |
|------|---------|
| `k8s/namespace.yaml` | Dev and Prod namespace definitions |
| `k8s/configmap.yaml` | Non-sensitive configuration |
| `k8s/secret.yaml` | Sensitive credentials (base64) |
| `k8s/postgres-deployment.yaml` | Database deployment + PVC |
| `k8s/backend-deployment.yaml` | Spring Boot backend (2 replicas) |
| `k8s/frontend-deployment.yaml` | React frontend (2 replicas) |
| `k8s/redis-deployment.yaml` | Redis cache deployment |
| `k8s/ingress.yaml` | Ingress routing rules |
| `k8s/hpa.yaml` | Horizontal Pod Autoscaler |

### Deployment Commands

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Deploy all resources
kubectl apply -f k8s/

# Verify deployment
kubectl get pods -n ecommerce-dev
kubectl get svc -n ecommerce-dev
kubectl describe pod <pod-name> -n ecommerce-dev

# Check logs
kubectl logs -f deployment/ecommerce-backend -n ecommerce-dev

# Port forward for local access
kubectl port-forward svc/ecommerce-frontend 8080:80 -n ecommerce-dev
```

### Minikube Local Development

```bash
# Start Minikube
minikube start --driver=docker --memory=4096 --cpus=2

# Enable ingress
minikube addons enable ingress

# Deploy application
kubectl apply -f k8s/

# Get Minikube IP
minikube ip

# Access application
minikube service ecommerce-frontend -n ecommerce-dev
```

---

## 15. Configuration Management (Ansible)

### Playbooks

| Playbook | Purpose |
|----------|---------|
| `ansible/playbook.yaml` | Main server configuration + Docker deployment |
| `ansible/k8s-deploy.yaml` | Kubernetes deployment automation |

### Inventory Configuration

```ini
# ansible/hosts.ini
[webservers]
web1 ansible_host=<EC2_IP> ansible_user=ubuntu

[kubernetes]
k8s-master ansible_host=<MASTER_IP> ansible_user=ubuntu
```

### Running Ansible

```bash
# Configure servers
cd ansible/
ansible-playbook -i hosts.ini playbook.yaml

# Deploy to Kubernetes
ansible-playbook -i hosts.ini k8s-deploy.yaml

# Dry run (check mode)
ansible-playbook -i hosts.ini playbook.yaml --check

# Verbose output
ansible-playbook -i hosts.ini playbook.yaml -vvv
```

---

## 16. Monitoring & Observability

### Stack Components

| Component | Purpose | Port |
|-----------|---------|------|
| Prometheus | Metrics collection | 9090 |
| Grafana | Visualization dashboards | 3000 |
| Node Exporter | Host metrics | 9100 |
| cAdvisor | Container metrics | 8080 |
| AlertManager | Alert handling | 9093 |

### Starting Monitoring Stack

```bash
cd monitoring/
docker-compose up -d

# Access Grafana: http://localhost:3000
# Username: admin | Password: admin123
```

### Key Metrics Monitored

- Request rate, Response time, Error rate
- CPU/Memory utilization
- Container health and restarts
- Database connections

---

## 17. Secret Management Strategy

| Environment | Storage | Method |
|-------------|---------|--------|
| Local Dev | `.env` file | Docker Compose |
| CI/CD | GitHub Secrets | Injected at runtime |
| Kubernetes | K8s Secrets | Environment variables |
| AWS | Secrets Manager | IAM role access |

---

## 18. Infrastructure Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Cloud (VPC)                          │
├─────────────────────────────────────────────────────────────────┤
│  Public Subnets          │         Private Subnets               │
│  ┌──────────────┐        │        ┌──────────────┐               │
│  │ NAT Gateway  │        │        │ EKS Nodes    │               │
│  │ Load Balancer│        │        │ ├─ Backend   │               │
│  └──────────────┘        │        │ ├─ Frontend  │               │
│         │                │        │ └─ Redis     │               │
│         ▼                │        └──────┬───────┘               │
│  ┌──────────────┐        │               │                       │
│  │ EKS Control  │◄───────┼───────────────┘                       │
│  └──────────────┘        │        ┌──────────────┐               │
│                          │        │ RDS Postgres │               │
│                          │        └──────────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

---

**Report Updated**: December 18, 2025  
**Lab Final Exam**: ✅ Complete  
