+++
categories = "terraform"
disableToc = true
title = "Securing RDS PostgreSQL with AWS Secrets Manager and Terraform"
weight = 20
tags = ["AWS", "Terraform", "RDS", "Secrets Manager", "PostgreSQL"]

+++

In this guide, we will explore how to securely manage PostgreSQL database credentials using AWS Secrets Manager when setting up an RDS PostgreSQL instance with Terraform. By storing credentials in Secrets Manager, we can enhance the security of our infrastructure.

### Why Use AWS Secrets Manager?

AWS Secrets Manager allows you to manage and retrieve database credentials, API keys, and other secrets throughout their lifecycle. It helps you to:

- **Securely store credentials:** Protect sensitive information with encryption.
- **Automatically rotate secrets:** Reduce the risk of leaked credentials.
- **Control access:** Fine-tune permissions to access secrets.

### Project Overview

Our project involves setting up an AWS infrastructure using Terraform with the following key components:

- **VPC:** Custom Virtual Private Cloud with public and private subnets.
- **S3 Bucket:** To store the Terraform state file.
- **DynamoDB Table:** For state locking to prevent concurrent modifications.
- **RDS PostgreSQL Instance:** Managed PostgreSQL database instance.
- **Secrets Manager Secret:** To securely store PostgreSQL credentials.
- **Security Group:** To control traffic to the RDS instance.
- **DB Subnet Group:** To define which subnets the RDS instance can use within the VPC.


#### Steps to Set Up AWS Secrets Manager with Terraform

##### Step 1: Store Credentials in Secrets Manager

First, we need to store the PostgreSQL credentials (username and password) in AWS Secrets Manager.

```sh
aws secretsmanager create-secret --name my-postgres-credentials --secret-string '{"username":"your-username","password":"your-password"}' --region ap-south-1
```

##### Step 2: Configure Terraform to Use Secrets Manager
In our Terraform configuration, we'll retrieve the secret value from AWS Secrets Manager.

```hcl
provider "aws" {
  region = "ap-south-1"
}

terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "path/to/your/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "your-dynamodb-lock-table"
  }
}

# Retrieve the secret value from AWS Secrets Manager
data "aws_secretsmanager_secret" "postgres_credentials" {
  name = "my-postgres-credentials"
}

data "aws_secretsmanager_secret_version" "postgres_credentials_version" {
  secret_id = data.aws_secretsmanager_secret.postgres_credentials.id
}

# Decode the secret value (JSON) and get the username and password
locals {
  secret_string = jsondecode(data.aws_secretsmanager_secret_version.postgres_credentials_version.secret_string)
}

# Create a VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway = true

  tags = {
    Name = "my-vpc"
  }
}

# Security Group for RDS
resource "aws_security_group" "rds_sg" {
  name_prefix = "rds_sg-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "rds_sg"
  }
}

# RDS PostgreSQL Instance
resource "aws_db_instance" "postgres" {
  allocated_storage    = 20
  engine               = "postgres"
  engine_version       = "13.3"
  instance_class       = "db.t3.micro"
  name                 = "mydb"
  username             = local.secret_string["username"]
  password             = local.secret_string["password"]
  db_subnet_group_name = aws_db_subnet_group.default.name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  skip_final_snapshot  = true

  tags = {
    Name = "my-postgres-db"
  }
}

# DB Subnet Group
resource "aws_db_subnet_group" "default" {
  name       = "main"
  subnet_ids = module.vpc.private_subnets

  tags = {
    Name = "main"
  }
}
```

##### Explanation

- **AWS Provider Configuration:** We specify the AWS region.
- **Secrets Manager Data Sources:** We use `data "aws_secretsmanager_secret"` and `data "aws_secretsmanager_secret_version"` to retrieve the secret and its version.
- **Local Variables:** We decode the secret string to extract the username and password.
- **RDS Resource Configuration:** We create an RDS PostgreSQL instance using the credentials stored in Secrets Manager.

##### Step 3: Deploy the Infrastructure

To deploy the infrastructure, run the following Terraform commands:

**Initialize the Terraform working directory:**

```sh
terraform init
```

Apply the configuration to create the resources:
```sh
terraform apply
```

This setup ensures that the PostgreSQL database credentials are securely managed using AWS Secrets Manager, enhancing the security and maintainability of our AWS infrastructure.

### Conclusion

Using AWS Secrets Manager with Terraform simplifies the management of sensitive credentials and improves the security of your infrastructure. By following this guide, you can securely set up an RDS PostgreSQL instance with managed credentials.

Feel free to reach out if you have any questions or need further assistance!

Happy Terraforming!