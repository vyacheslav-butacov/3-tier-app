## 1. Infrastructure Platform Selection

**Chosen Platform:** **Amazon Web Services (AWS)**

**Arguments:**

- **Comprehensive Services:** AWS offers a wide range of services that cater to all the requirements, including compute, storage, databases, networking, and security.
- **Scalability:** AWS services like EC2, ECS, and EKS provide robust autoscaling capabilities.
- **High Availability:** AWS's global infrastructure ensures fault tolerance and high availability through multiple Availability Zones (AZs) and Regions.
- **Security:** AWS provides extensive security features, including IAM, VPC, Security Groups, and encryption services.
- **Automation Support:** Tools like AWS CloudFormation and Terraform facilitate infrastructure as code (IaC) for automated deployments.



## 2. Orchestration Technology and Components

**Chosen Orchestration Technology:** **Kubernetes (EKS on AWS)**

**Components:**

- **Amazon EKS (Elastic Kubernetes Service):** Managed Kubernetes service to orchestrate containerized microservices.
- **Helm:** Package manager for Kubernetes to manage complex applications.
- **Ingress Controller (NGINX):** Manages external access to services within the cluster.
- **Cert-Manager:** Automates the management and issuance of TLS certificates.
- **Prometheus & Grafana:** For monitoring and visualization.
- **Kube-Proxy and CNI (e.g., Calico):** For networking within the cluster.

**Arguments:**

- **Scalability:** Kubernetes inherently supports scaling of applications based on demand.
- **Fault Tolerance:** Kubernetes ensures that applications are running as desired, automatically replacing failed containers.
- **Extensibility:** A rich ecosystem of tools and plugins enhances Kubernetes capabilities.
- **Community Support:** Strong community support ensures continuous improvements and best practices.



## 3. Automated Infrastructure Deployment

**Solution:**

Utilize **Terraform** for infrastructure as code to automate the provisioning of AWS resources required for the **openinnovation.ai** platform.

**Key Components to Automate:**

- **VPC Setup:** Networking configuration including subnets, route tables, and internet gateways.
- **EKS Cluster:** Managed Kubernetes cluster setup.
- **RDS for PostgreSQL:** Managed relational database service.
- **IAM Roles and Policies:** Secure access control.
- **Autoscaling Groups:** For EC2 instances if needed.

**Important Configuration Snippets:**

```hcl
# infra/main.tf

provider "aws" {
  region = " me-central-1"
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = "openinnovation-ai-vpc"
  cidr   = "10.0.0.0/16"

  azs             = ["me-central-1a", "me-central-1b", "me-central-1c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]

  enable_nat_gateway = true
  one_nat_gateway_per_az = true
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "openinnovation-ai-cluster"
  cluster_version = "1.30"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id

  worker_groups_launch_template = [
    {
      instance_type = "m6i.large" # last generation for me-central-1 region
      asg_max_size  = 5
      asg_min_size  = 2
    }
  ]
}

resource "aws_db_instance" "postgres" {
  identifier              = "openinnovation-ai-postgres-db"
  engine                  = "postgres"
  instance_class          = "db.t4g.medium"
  allocated_storage       = 20
  name                    = "openinnovationaidb"
  username                = jsondecode(data.aws_secretsmanager_secret_version.postgres_credentials_version.secret_string).username
  password                = jsondecode(data.aws_secretsmanager_secret_version.postgres_credentials_version.secret_string).password
  vpc_security_group_ids  = [module.vpc.default_security_group_id]
  db_subnet_group_name    = module.vpc.database_subnets
  multi_az                = true
  publicly_accessible     = false
  storage_encrypted       = true
  backup_retention_period = 7
}
```


## 4. Automated Microservices Deployment

**Solution:**

Implement **CI/CD pipelines** using **GitHub Actions** to automate the build, test, and deployment of microservices to the Kubernetes cluster for **openinnovation.ai**.

## Key Steps

1. **Build Docker Images:** 
   - Build Docker images for both backend and frontend services.
   - Push the built images to a container registry (e.g., Amazon ECR).

2. **Deploy to Kubernetes:** 
   - Use `kubectl` or Helm charts to deploy the services to the EKS cluster.
   - Ensure that the deployments are updated with the latest image tags.

3. **Manage Configurations and Secrets:**
   - Securely manage configurations and secrets by fetching them from AWS Secrets Manager.
   - Integrate Kubernetes with external secrets management tools to inject secrets into the deployments.

## Important Configuration Snippets

### GitHub Actions Workflow

Create a GitHub Actions workflow to handle the CI/CD process. This workflow will build Docker images, push them to Amazon ECR, and update the Kubernetes deployments.

```yaml
# .github/workflows/deploy.yml

name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build and Push Backend Image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/backend:${IMAGE_TAG} ./backend
          docker push $ECR_REGISTRY/backend:${IMAGE_TAG}

      - name: Build and Push Frontend Image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/frontend:${IMAGE_TAG} ./frontend
          docker push $ECR_REGISTRY/frontend:${IMAGE_TAG}

      - name: Configure kubectl
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update Kubernetes Deployment
        run: |
          kubectl set image deployment/backend-deployment backend=${{ steps.login-ecr.outputs.registry }}/backend:${{ github.sha }}
          kubectl set image deployment/frontend-deployment frontend=${{ steps.login-ecr.outputs.registry }}/frontend:${{ github.sha }}
```

### Helm Deployment Manifests
Define Kubernetes deployment manifests using Helm to manage the backend and frontend services. These manifests will reference the Docker images pushed to ECR and utilize secrets for sensitive information.
```
# microservices/helm/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: <ECR_REGISTRY>/backend:latest
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DATABASE_URL
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: <ECR_REGISTRY>/frontend:latest
          ports:
            - containerPort: 80
```



### External Secrets Configuration
To securely access PostgreSQL credentials stored in AWS Secrets Manager, use tools like External Secrets or Secrets Store CSI Driver. Below is an example using External Secrets.
```
# microservices/helm/postgres-secret.yaml

apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: openinnovation-ai-postgres-credentials
        property: username
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: openinnovation-ai-postgres-credentials
        property: password
```

### Explanation
- **CI/CD Pipeline**: The GitHub Actions workflow automates the entire process of building, testing, and deploying microservices. It ensures that every commit to the main branch triggers the pipeline, promoting continuous integration and delivery.

- **Docker Image Management**: By tagging Docker images with the commit SHA, you ensure traceability and uniqueness of each build. Pushing these images to Amazon ECR provides a secure and scalable container registry.

- **Kubernetes Deployment**: Using kubectl set image, the workflow updates the running deployments with the new Docker images. Helm charts manage the Kubernetes resources, promoting consistency and reusability.

- **Secrets Management**: Integrating External Secrets with AWS Secrets Manager ensures that sensitive information like database credentials are securely managed and injected into the Kubernetes environment without hardcoding them into manifests or code.

- **Security**: By leveraging AWS IAM roles and policies, the system ensures that only authorized components can access the necessary secrets, adhering to the principle of least privilege.

- **Scalability and Reliability**: Deploying multiple replicas for each service ensures high availability and load balancing. Kubernetes handles the orchestration, scaling, and self-healing of the microservices.



# 5. Release Lifecycle for Components

## Overview

Establish a streamlined release lifecycle to ensure seamless deployment, maintenance, and scalability for **openinnovation.ai** microservices.

## Backend Service

1. **Development**
   - Commit code to the `backend` repository.
   
2. **Continuous Integration (CI)**
   - Run unit tests and static code analysis.
   
3. **Build Process**
   - Build and tag Docker images.
   - Push images to Amazon ECR.
   
4. **Continuous Deployment (CD)**
   - Deploy to staging for integration testing.
   
5. **Production Deployment**
   - Upon approval, deploy to production using Helm.
   - Verify deployment success.

## Frontend Service

1. **Development**
   - Commit code to the `frontend` repository.
   
2. **Continuous Integration (CI)**
   - Run linting and tests.
   
3. **Build Process**
   - Build and tag Docker images.
   - Push images to Amazon ECR.
   
4. **Continuous Deployment (CD)**
   - Deploy to staging for user acceptance testing.
   
5. **Production Deployment**
   - Upon approval, deploy to production using Helm.
   - Verify deployment success.

## PostgreSQL Database

1. **Provisioning**
   - Managed via Terraform.
   
2. **Schema Migrations**
   - Use Flyway or Liquibase integrated into the CI/CD pipeline.
   
3. **Backup & Restore**
   - Automated backups with AWS RDS.

## Release Management

- **Versioning:** Use semantic versioning (e.g., v1.0.0) for all services.
- **Rollback Strategy:** Utilize Kubernetes rolling updates to facilitate easy rollbacks in case of issues.
- **Approval Gates:** Implement manual approval steps in the CI/CD pipeline for production deployments to ensure quality control.




# 6. Testing Approach for Infrastructure
## Strategy:

- **Unit** Testing: Test individual Terraform modules using tools like terraform validate and tflint.
- **Integration Testing**: Deploy infrastructure to a test environment and verify resources using tools like Terratest.
- **Compliance Testing**: Ensure infrastructure adheres to security and compliance standards using tools like Checkov or AWS Config.
- **Continuous Testing**: Integrate tests into the CI/CD pipeline to automatically validate infrastructure changes.

```
# Run Terraform validation and linting
terraform init
terraform validate
tflint
```
- **Add Testing Steps to GitHub Actions**
```
# .github/workflows/ci.yml

name: Terraform CI

on:
  push:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.9.5

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Run TFLint
        run: tflint

      - name: Run Checkov
        run: checkov -d infra/

```