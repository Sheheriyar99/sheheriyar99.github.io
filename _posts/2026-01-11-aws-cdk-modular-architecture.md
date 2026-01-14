l1---
layout: post
title: "Building Scalable AWS Infrastructure with CDK: A Modular Stack Architecture"
date: 2025-11-15
categories: [AWS, CDK, Infrastructure, Architecture]
tags: [aws-cdk, infrastructure-as-code, devops, architecture]
---

## Introduction

Infrastructure as Code (IaC) has become essential for managing cloud resources effectively. In this guide, I'll walk you through how I designed a scalable and maintainable AWS infrastructure using AWS CDK with a modular stack architecture. This approach separates concerns, improves reusability, and makes managing complex deployments straightforward.

## The Problem We're Solving

When building cloud applications with multiple components—databases, application servers, load balancers, and monitoring—you face several challenges:

1. **Tight Coupling**: Components become interdependent, making changes risky
2. **Difficult Testing**: Hard to test infrastructure changes in isolation
3. **Scaling Issues**: Replicating the same infrastructure for different environments is repetitive
4. **Clear Dependencies**: Managing relationships between resources becomes complex

## The Solution: Three-Stack Architecture

I designed the infrastructure as three separate but interdependent CDK stacks:

```
CoreInfraStack
├── VPC
├── ECR Repository
├── Security Groups (ALB, ECS, RDS)
└── Network Configuration

AppInfraStack (depends on CoreInfraStack)
├── Application Load Balancer
├── ECS Cluster
├── ECS Service & Task Definition
├── IAM Roles
└── CloudWatch Integration

DbInfraStack (depends on CoreInfraStack)
├── Aurora PostgreSQL Cluster
├── RDS Instances (Writer + Readers)
├── Database Credentials & Secrets
└── Backup & Retention Policies
```

## Architecture Breakdown

### 1. CoreInfraStack - The Foundation Layer

This stack creates the foundational networking and security infrastructure:

**Key Components:**
```typescript
- VPC with configurable CIDR (10.1.0.0/16)
- Public subnets across multiple AZs
- ECR Repository for Docker images
- Three security groups:
  - ALB Security Group (for load balancer ingress)
  - ECS Security Group (for container traffic)
  - RDS Security Group (for database access)
```

**Why Separate This:**
- Networking rarely changes and can be reused across multiple application stacks
- Security groups are foundational for all downstream resources
- Creates a clean boundary between infrastructure and application layers

### 2. AppInfraStack - The Application Layer

This stack deploys the containerized application infrastructure:

**Key Components:**
```typescript
- Application Load Balancer
- Target Groups
- ECS Cluster
- Fargate Services
- Task Definitions with environment configuration
- IAM roles for application permissions
```

**Why This Design:**
- Decoupled from database concerns
- Can be deployed independently after CoreInfraStack
- Easily scalable to multiple application services
- Configuration driven (uses config.json for customization)

### 3. DbInfraStack - The Data Layer

This stack manages all database infrastructure:

**Key Components:**
```typescript
- Aurora PostgreSQL cluster
- Reader and writer instances (r6i.large)
- Automatic backups (30-day retention)
- Performance Insights enabled
- Publicly accessible for development
```

**Why Separate:**
- Database changes can be managed independently
- Easier to scale read replicas without touching application
- Can be deployed on a different schedule
- Retention policies (deletion protection) protect critical data

## Data Flow & Stack Dependencies

```
┌─────────────────────────────────────────┐
│          CoreInfraStack                 │
│  (VPC, Security Groups, ECR)            │
└──────────────┬──────────────────────────┘
               │
        ┌──────┴──────┐
        │             │
┌───────▼────────┐  ┌─▼──────────────┐
│  AppInfraStack │  │ DbInfraStack   │
│  (ECS, ALB)    │  │ (RDS Aurora)   │
└────────────────┘  └────────────────┘
        │                   │
        └───────┬───────────┘
                │
        Application Running
```

## Configuration-Driven Approach

The infrastructure uses a `config.json` file for environment-specific settings:

```json
{
  "PROJECT_NAME": "mainapp",
  "ENVIRONMENT": "dev",
  "VPC_IP": "10.1.0.0/16",
  "db": {
    "username": "dbmasteruser",
    "password": "***",
    "name": "applicationDB",
    "port": "5432"
  },
  "mainapp-svcTask": {
    "cpu": 4096,
    "memory": 12288,
    "desiredCount": 1
  }
}
```

**Benefits:**
- Same CDK code works for dev, staging, and production
- Easy to adjust resources without changing code
- Version control friendly (exclude password in actual setup)
- Clear documentation of all configuration parameters

## Key Design Patterns Used

### 1. **Stack Exports for Inter-Stack Communication**

Instead of hardcoding values, we use CloudFormation exports:

```typescript
new cdk.CfnOutput(this, "VPC_ID", { 
  value: this.vpc.vpcId,
  exportName: this.getLogicalEnvName("vpc-id")
});
```

**Why:** Decouples stacks, allows independent updates, follows AWS best practices.

### 2. **Security Group Chain**

```
ALB SG ──ingress──> ECS SG ──ingress──> RDS SG
  (port 80)          (port 80)          (port 5432)
```

**Why:** Fine-grained access control, only necessary ports open, defense-in-depth.

### 3. **Environment Variable Injection**

Task definitions receive configuration through environment variables:

```typescript
envVars["DB_HOST"] = config.get("db.endpoint");
envVars["APP_ENV"] = appConfig.appEnv;
// ... etc
```

**Why:** Single image, multiple environments; secure (uses secrets manager for sensitive data).

### 4. **Role-Based IAM**

Separated IAM roles:
- **Execution Role**: Pulls images, writes logs (fixed permissions)
- **Task Role**: Application permissions (customizable per service)

**Why:** Principle of least privilege, security best practice.

## Deployment Process

```bash
# Deploy all stacks with dependencies automatically managed
cdk deploy --all

# Or deploy individual stacks
cdk deploy CoreInfraStack
cdk deploy DbInfraStack
cdk deploy AppInfraStack
```

CDK automatically handles:
1. Deploying in correct order (CoreInfraStack first)
2. Waiting for stack completion
3. Passing outputs between stacks
4. Rollback on failure

## Real-World Benefits I've Experienced

### 1. **Easy Environment Replication**
- Copied config.json, deployed to production with zero code changes
- Confidence that production mirrors development setup

### 2. **Safe Updates**
- Update AppInfraStack without touching database
- Update DbInfraStack with deletion protection enabled
- Database changes don't impact running applications

### 3. **Clear Ownership**
- Database team owns DbInfraStack
- Application team owns AppInfraStack
- Platform team owns CoreInfraStack
- Each can work independently

### 4. **Debugging Made Easy**
- Export values from each stack for inspection
- Security groups clearly show traffic flow
- CloudWatch logs per service
- CloudFormation console shows exact resources created

### 5. **Cost Optimization**
- Easy to adjust CPU/memory in configuration
- Can disable/enable features without code changes
- Backup retention policies configurable per environment

## Advanced Features Included

### Monitoring & Observability
- CloudWatch Logs for ECS tasks (3-day retention)
- CloudWatch Alarms for CPU/Memory (commented, ready to enable)
- Performance Insights on RDS instances

### High Availability
- Multi-AZ deployment (2 availability zones)
- Aurora reader replicas
- Load balancing across instances

### Security
- Deletion protection on RDS
- Encryption at rest on database
- Security group-based access control
- IAM role-based permissions

## Performance Considerations

**Instance Selection:**
```
ECS Tasks: Fargate (serverless)
- No EC2 management overhead
- Pay per vCPU/GB second used
- Auto-scaling capable

RDS: r6i.large instances
- Memory optimized
- Good for database workloads
- Cost-effective for this scale
```

## Lessons Learned

1. **Use Logical Names**: Consistent naming (`${PROJECT_NAME}-${ENVIRONMENT}-${RESOURCE_TYPE}`) prevents confusion

2. **Export Everything**: CloudFormation exports create discoverable outputs for cross-stack references

3. **Configuration Matters**: Even small logic changes warrant environment-specific config

4. **Security Groups Are Critical**: Properly configured security groups prevent many production issues

5. **Test Dependencies**: Always test stack deletion in reverse order to verify dependency configuration

## Extending This Architecture

This foundation easily supports:

- **Multi-service deployments**: Add more AppInfraStack instances for different services
- **Multi-region**: Deploy entire stack setup in different regions
- **Blue-Green Deployments**: Create parallel stacks, route traffic via Route53
- **Cost allocation**: Tags automatically propagate to all resources

## Conclusion

By separating concerns into three focused stacks, this architecture provides:

✅ **Modularity** - Each stack has a single responsibility  
✅ **Reusability** - CoreInfraStack shared across environments  
✅ **Maintainability** - Changes isolated to relevant stack  
✅ **Scalability** - Easy to add services and features  
✅ **Safety** - Dependencies clearly defined, rollback automatic  

This approach has served us well for production applications and I recommend it for anyone building non-trivial infrastructure on AWS.

---

**Have questions about this architecture? Feel free to reach out on [GitHub](https://github.com/Sheheriyar99) or [LinkedIn](https://www.linkedin.com/in/shaheryar-99s)!**
