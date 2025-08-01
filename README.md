# NocoDB AWS Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS CLOUD                                        │
│  ┌───────────────────────────────────────────────────────────────────────────────── │
│  │                                   VPC (10.0.0.0/16)                            | │
│  │                                                                                | │
│  │  ┌──────────────────────────┐         ┌──────────────────────────┐             │ │
│  │  │      Public Subnet       │         │      Public Subnet       │             │ │
│  │  │    (10.0.1.0/24)         │         │    (10.0.2.0/24)         │             │ │
│  │  │         AZ-1a            │         │         AZ-1b            │             │ │
│  │  │                          │         │                          │             │ │
│  │  │  ┌─────────────────────┐ │         │ ┌─────────────────────┐  │             │ │
│  │  │  │    NAT Gateway      │ │         │ │    NAT Gateway      │  │             │ │
│  │  │  │     + EIP           │ │         │ │     + EIP           │  │             │ │
│  │  │  └─────────────────────┘ │         │ └─────────────────────┘  │             │ │
│  │  └──────────────────────────┘         └──────────────────────────┘             │ │
│  │                    │                              │                            │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    Application Load Balancer                            │   │ │
│  │  │                       (Internet-facing)                                 │   │ │
│  │  │                      Port 80 → Port 8080                                │   │ │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │ │
│  │                    │                              │                            │ │
│  │  ┌──────────────────────────┐         ┌──────────────────────────┐             │ │
│  │  │     Private Subnet       │         │     Private Subnet       │             │ │
│  │  │    (10.0.11.0/24)        │         │    (10.0.12.0/24)        │             │ │
│  │  │         AZ-1a            │         │         AZ-1b            │             │ │
│  │  │                          │         │                          │             │ │
│  │  │ ┌──────────────────────┐ │         │ ┌──────────────────────┐ │             │ │
│  │  │ │   EC2 Instance       │ │         │ │   EC2 Instance       │ │             │ │
│  │  │ │  (NocoDB App)        │ │         │ │  (NocoDB App)        │ │             │ │
│  │  │ │   t2.micro           │ │         │ │   t2.micro           │ │             │ │
│  │  │ │   Port: 8080         │ │         │ │   Port: 8080         │ │             │ │
│  │  │ └──────────────────────┘ │         │ └──────────────────────┘ │             │ │
│  │  └──────────────────────────┘         └──────────────────────────┘             │ │
│  │                    │                              │                            │ │
│  │                    └──────────────┬───────────────┘                            │ │
│  │                                   │                                            │ │
│  │  ┌──────────────────────────┐         ┌──────────────────────────┐             │ │
│  │  │    Database Subnet       │         │    Database Subnet       │             │ │
│  │  │    (10.0.21.0/24)        │         │    (10.0.22.0/24)        │             │ │
│  │  │         AZ-1a            │         │         AZ-1b            │             │ │
│  │  │                          │         │                          │             │ │
│  │  │ ┌──────────────────────┐ │         │ ┌──────────────────────┐ │             │ │
│  │  │ │   RDS PostgreSQL     │ │         │ │   ElastiCache        │ │             │ │
│  │  │ │   (Multi-AZ)         │ │         │ │   Redis Cluster      │ │             │ │
│  │  │ │   db.t3.micro        │ │         │ │   cache.t3.micro     │ │             │ │
│  │  │ │   Port: 5432         │ │         │ │   Port: 6379         │ │             │ │
│  │  │ │   Encrypted          │ │         │ │   Encrypted          │ │             │ │
│  │  │ └──────────────────────┘ │         │ └──────────────────────┘ │             │ │
│  │  └──────────────────────────┘         └──────────────────────────┘             │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              External Services                                  │ │
│  │                                                                                 │ │
│  │  ┌──────────────────────┐              ┌──────────────────────┐                 │ │
│  │  │  Systems Manager     │              │   Internet Gateway   │                 │ │
│  │  │  Parameter Store     │              │                      │                 │ │
│  │  │  (DB Password)       │              │                      │                 │ │
│  │  └──────────────────────┘              └──────────────────────┘                 │ │
│  └─────────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘

Security Groups:
┌─────────────────────┬─────────────────┬──────────────────────────────────────────┐
│    Resource         │      Port       │              Access Rules                │
├─────────────────────┼─────────────────┼──────────────────────────────────────────┤
│ ALB                 │ 80, 443         │ Internet (0.0.0.0/0)                     │
│ EC2 (NocoDB)        │ 8080            │ ALB Security Group only                  │
│ EC2 (SSH)           │ 22              │ Internet (⚠️  Restrict in production)    │
│ RDS PostgreSQL      │ 5432            │ EC2 Security Group only                  │
│ ElastiCache Redis   │ 6379            │ EC2 Security Group only                  │
└─────────────────────┴─────────────────┴──────────────────────────────────────────┘

Key Features:
• Multi-AZ deployment for high availability
• Auto Scaling Group with Launch Template
• Encrypted storage (EBS + RDS + ElastiCache)
• Private subnets for application and database tiers
• NAT Gateways for outbound internet access from private subnets
• Secrets management via AWS Systems Manager Parameter Store
• Load balancer health checks with proper NocoDB endpoints
• IAM roles with least privilege access for EC2 instances

Flow:
Internet → ALB (Port 80) → EC2 NocoDB (Port 8080) → RDS PostgreSQL (Port 5432)
                                                   → ElastiCache Redis (Port 6379)
```

## Resource Overview

| Component | Type | Size | Purpose |
|-----------|------|------|---------|
| **VPC** | Virtual Private Cloud | 10.0.0.0/16 | Network isolation |
| **Subnets** | Public/Private/DB | /24 each | Network segmentation |
| **ALB** | Application Load Balancer | - | Traffic distribution |
| **EC2** | Auto Scaling Group | t2.micro (Free Tier) | NocoDB application |
| **RDS** | PostgreSQL | db.t3.micro (Free Tier) | Primary database |
| **ElastiCache** | Redis | cache.t3.micro (Free Tier) | Caching layer |
| **NAT Gateway** | Managed NAT | 2x Multi-AZ | Outbound internet access |

## Deployment Commands

```bash
# Make the script executable
chmod +x create.sh

# Run the script to generate project structure
./create.sh

# Navigate to project directory
cd nocodb-aws-terraform-modular

# Initialize Terraform
terraform init

# Review the deployment plan
terraform plan

# Deploy the infrastructure
terraform apply
```

**Note**: Remember to create an EC2 Key Pair in AWS Console before running `terraform apply` and update the `key_pair_name` variable accordingly.
