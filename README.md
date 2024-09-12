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