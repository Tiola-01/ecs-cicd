# AWS ECS Fargate CI/CD Pipeline

A production-oriented CI/CD project demonstrating container deployment to Amazon ECS Fargate using Jenkins, Docker, Amazon ECR, Terraform, and an Application Load Balancer.

## Architecture

GitHub → Jenkins → Docker → Amazon ECR → ECS Fargate → ALB

## Infrastructure

Terraform provisions:

- AWS VPC
- Public and private subnets across two Availability Zones
- Internet Gateway
- NAT Gateway
- Security Groups
- Amazon ECR
- ECS Cluster
- ECS Fargate Service
- Application Load Balancer
- CloudWatch Logs
- IAM ECS Task Execution Role

## CI/CD Pipeline

Jenkins:

1. Checks out application code from GitHub
2. Builds the Docker image
3. Authenticates to Amazon ECR
4. Pushes an immutable versioned image
5. Registers a new ECS task definition revision
6. Updates the ECS service
7. Waits for ECS deployment stabilization

## Security

- ECS tasks run in private subnets
- ALB is publicly accessible
- ECS security group only accepts HTTP from the ALB
- ECR image tags are immutable
- ECR scanning is enabled
- AWS credentials are stored in Jenkins credentials
- Terraform state is stored remotely in Amazon S3

## Validation

The pipeline was validated by deploying application Version 2.0 through:

GitHub → Jenkins → Docker → ECR → ECS Fargate → ALB

The ECS service successfully maintained two running tasks and reached a stable deployment.

## Production Evolution

Future improvements:

- HTTPS with ACM
- ECS Service Auto Scaling
- Multi-AZ NAT Gateways
- Jenkins IAM role / OIDC instead of long-lived access keys
- AWS Secrets Manager
- Blue/green deployments
- CloudWatch alarms
- Automated rollback
