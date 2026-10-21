# .github-workflows-pr-checks.yml name: Secure Terraform PR Checks

on:
  pull_request:
    branches:
      - main

permissions:
  contents: read
  id-token: write

jobs:
  terraform-checks:
    name: Terraform Validation and Plan
iam.tf
https://token.actions.githubusercontent.com
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Initialize
        run: terraform init -backend=false

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -input=false -no-color 

versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

Terraform version and AWS provider configuration

variables.tf
variable "aws_region" {
  description = "AWS region for the infrastructure"
  type        = string
  default     = "eu-west-1"
}

variable "github_repository" {
  description = "GitHub repository in owner/repository format"
  type        = string
}

variable "github_branch" {
  description = "Branch permitted to assume the deployment role"
  type        = string
  default     = "main"
}

variable "frontend_bucket_name" {
  description = "Globally unique S3 bucket name for the frontend"
  type        = string
}

variable "app_runner_image_identifier" {
  description = "Container image URI for the backend"
  type        = string
}
 Terraform input variables
Add GitHub OIDC provider and 
IAM role