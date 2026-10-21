
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
backend.tf

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
frontend.tf

resource "aws_s3_bucket" "frontend" {
  bucket = var.frontend_bucket_name
}

resource "aws_s3_bucket_public_access_block" "frontend" {
  bucket = aws_s3_bucket.frontend.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_cloudfront_origin_access_control" "frontend" {
  name                              = "frontend-oac"
  description                       = "Secure access to the frontend S3 bucket"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "frontend" {
  enabled             = true
  default_root_object = "index.html"

  origin {
    domain_name              = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id                = "frontend-s3-origin"
    origin_access_control_id = aws_cloudfront_origin_access_control.frontend.id
  }

  default_cache_behavior {
    target_origin_id       = "frontend-s3-origin"
    viewer_protocol_policy = "redirect-to-https"

    allowed_methods = ["GET", "HEAD"]
    cached_methods  = ["GET", "HEAD"]

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

data "aws_iam_policy_document" "frontend_bucket" {
  statement {
    sid    = "AllowCloudFrontReadOnly"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    actions = [
      "s3:GetObject"
    ]

    resources = [
      "${aws_s3_bucket.frontend.arn}/*"
    ]

    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.frontend.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "frontend" {
  bucket = aws_s3_bucket.frontend.id
  policy = data.aws_iam_policy_document.frontend_bucket.json

  depends_on = [
    aws_s3_bucket_public_access_block.frontend
  ]
}
Add secure S3 and CloudFront frontend
backend.tf
resource "aws_apprunner_service" "backend" {
  service_name = "secure-ci-backend"

  source_configuration {
    auto_deployments_enabled = true

    image_repository {
      image_identifier      = var.app_runner_image_identifier
      image_repository_type = "ECR"

      image_configuration {
        port = "8080"

        runtime_environment_variables = {
          PORT = "8080"
        }
      }
    }
  }

  health_check_configuration {
    protocol            = "HTTP"
    path                = "/health"
    interval            = 10
    timeout             = 5
    healthy_threshold   = 1
    unhealthy_threshold = 5
  }

  instance_configuration {
    cpu    = "1 vCPU"
    memory = "2 GB"
  }

  tags = {
    Project     = "Secure-CI-Pipeline"
    Environment = "Development"
  }
}

AWS APP runner backend infrastructure 
Outputs.tf
output "frontend_cloudfront_url" {
  description = "CloudFront URL for the static frontend"
  value       = "https://${aws_cloudfront_distribution.frontend.domain_name}"
}

output "backend_service_url" {
  description = "AWS App Runner backend URL"
infrastructure output 
  value       = aws_apprunner_service.backend.service_url
}

output "github_actions_role_arn" {
  description = "IAM role ARN for GitHub Actions"
  value       = aws_iam_role.github_actions.arn
}
outputs.tf.

## Secure CI Pipeline with GitHub OIDC

### Project Overview

This project uses Terraform and GitHub Actions to validate infrastructure changes securely. The infrastructure includes a private Amazon S3 bucket with a CloudFront distribution for the frontend and an AWS App Runner service for the backend.

### GitHub OIDC Authentication

GitHub Actions authenticates with AWS using OpenID Connect (OIDC). An AWS IAM OIDC identity provider trusts tokens issued by GitHub Actions.

The IAM role trust policy restricts access to the specified GitHub repository and main branch using the token's subject (`sub`) condition. The role is intended to use least-privilege permissions rather than AdministratorAccess.

No permanent AWS access keys are stored in the GitHub workflow.

### Pull Request Workflow

The workflow is defined in `.github/workflows/pr-checks.yml`.

When a pull request targets the main branch, the workflow runs:

- `terraform fmt -check`
- `terraform init`
- `terraform validate`
- `terraform plan`

The workflow does not run `terraform apply`. This ensures that pull requests can validate and preview infrastructure changes without automatically modifying live infrastructure.

Infrastructure deployment should be handled separately through a controlled workflow, preferably with approval before applying changes.

### Frontend Infrastructure

The frontend uses an Amazon S3 bucket protected by S3 Block Public Access. Amazon CloudFront serves the frontend over HTTPS using Origin Access Control (OAC), restricting direct access to the bucket.

### Backend Infrastructure

The backend uses AWS App Runner to run a container image from Amazon ECR. The service is configured with a health check endpoint and automatic deployment settings.

Application secrets should be supplied through an appropriate managed configuration rather than hardcoded into the container image.

### Stretch Goals

- OIDC authentication instead of long-lived AWS access keys.
- Least-privilege IAM role design.
- Private S3 frontend origin using CloudFront OAC.
- HTTPS through CloudFront.
- App Runner automatic deployment configuration.
- Separate planning and deployment workflows.

### Security Notes

The pull request workflow must not give untrusted pull requests AWS credentials capable of modifying infrastructure. AWS permissions should be restricted to the operations required by the pipeline.

Terraform deployment should be performed separately from pull request validation, with appropriate access controls and approval.
Document secure CI Pipeline and OIDC

the IAM permissions required by Terraform.

Ensure the OIDC trust policy and workflow are compatible and secure.
Configure the GitHub repository variables.
Check the App Runner ECR access configuration.
Run Terraform formatting, initialization,validation,and planning.
Ensure the workflow publishes or uploads the plan output.