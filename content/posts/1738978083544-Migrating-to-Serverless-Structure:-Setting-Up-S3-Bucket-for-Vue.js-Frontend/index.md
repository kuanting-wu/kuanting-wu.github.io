---
title: "Migrating to Serverless Structure: Setting Up S3 Bucket for Vue.js Frontend"
date: 2025-02-08
draft: false
description: "a description"
tags: ["S3", "Terraform", "IAM", "Github Action"]
---

This post will explore how to set up an S3 bucket for hosting a Vue.js frontend as part of migrating to a serverless structure. Covering Terraform configurations and key considerations for proper routing, followed by setting up OpenID Connect in IAM roles, policies, and GitHub Actions permissions.

## Creating Terraform Code for S3 Bucket
To streamline infrastructure setup, I use Terraform to define and deploy resources. The infrastructure code for this setup can be found [Here](https://github.com/kuanting-wu/qj_infra/blob/main/s3.tf).

Below is an example Terraform configuration to create an S3 bucket for static website hosting:

```hcl
# S3 bucket for hosting a Vue.js website
resource "aws_s3_bucket" "vue_website" {
  bucket = "quantifyjiujitsu.com" # Must be globally unique

  tags = {
    Name = "Quantify Jiujitsu Website Bucket"
  }
}

resource "aws_s3_bucket_public_access_block" "public_access_block" {
  bucket                  = aws_s3_bucket.vue_website.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}


# Bucket policy to allow public access to all objects within the bucket
resource "aws_s3_bucket_policy" "vue_policy" {
  bucket = aws_s3_bucket.vue_website.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.vue_website.arn}/*"
      }
    ]
  })
  depends_on = [aws_s3_bucket_public_access_block.public_access_block]
}

resource "aws_s3_bucket_website_configuration" "example" {
  bucket = aws_s3_bucket.vue_website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "index.html"
  }
}
```

### Important Configuration Note
Since Vue.js is a single-page application, proper routing configuration is essential. Ensure both the `index_document` and `error_document` are set to `index.html`. This ensures all routes are correctly handled by the frontend.

## Setting Up OIDC for Secure GitHub Actions Integration

To securely deploy infrastructure using GitHub Actions, we recommend setting up OIDC (OpenID Connect) authentication with AWS. This approach eliminates the need to store long-term credentials.

### Terraform Configuration for IAM Setup

The following Terraform code snippet from the [repository](https://github.com/kuanting-wu/qj_infra/blob/main/s3.tf) sets up the necessary IAM components for OIDC integration:

```hcl
data "aws_caller_identity" "current" {}

resource "aws_iam_openid_connect_provider" "github_actions_oidc" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

resource "aws_iam_role" "github_oidc_role" {
  name = "github-actions-oidc-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/token.actions.githubusercontent.com"
        },
        Action    = "sts:AssumeRoleWithWebIdentity",
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com",
            "token.actions.githubusercontent.com:sub" = "repo:${var.github_repo}:ref:refs/heads/main"
          }
        }
      }
    ]
  })
}

resource "aws_iam_policy" "github_deploy_policy" {
  name        = "github-deploy-s3-policy"
  description = "Policy for GitHub Actions to deploy to S3"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect   = "Allow",
        Action   = ["s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
        Resource = [
          "arn:aws:s3:::${var.s3_bucket_name}",
          "arn:aws:s3:::${var.s3_bucket_name}/*"
        ]
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "github_deploy_policy_attach" {
  role       = aws_iam_role.github_oidc_role.name
  policy_arn = aws_iam_policy.github_deploy_policy.arn
}
```

### Setting Up GitHub Actions

Sample GitHub Actions Workflow:

```yaml
name: Deploy Vue.js to S3

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Build the Vue.js project
        run: npm run build

      - name: Configure AWS CLI
        uses: aws-actions/configure-aws-credentials@v3
        with:
          role-to-assume: arn:aws:iam::654654621599:role/${{ secrets.AWS_GITHUB_ROLE }}
          role-session-name: github-action
          aws-region: us-east-1

      - name: Deploy to S3
        run: aws s3 sync dist/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
```

### Note for GitHub Secrets

To securely manage sensitive information, the following GitHub secrets are required:

1. `secrets.AWS_GITHUB_ROLE`
   - This secret should contain the ARN of the IAM role that GitHub Actions will assume during deployment.
   - Example value: `arn:aws:iam::654654621599:role/github-actions-oidc-role`

2. `secrets.S3_BUCKET_NAME`
   - This secret should hold the name of the S3 bucket where the Vue.js frontend will be deployed.
   - Example value: `qj-frontend-bucket`