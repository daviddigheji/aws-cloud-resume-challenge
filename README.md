# cloud-resume Challenge - David Digheji

## overview
This mini project is a fully serverless resume build using AWS.
it uses Cloud Engineering, Devops, and infrusture automation skillset.

## live website
https://daviddigheji.com

##  Architecture
User → CloudFront → S3
                 ↓
            API Gateway → Lambda → DynamoDB

## Technologies Used
- AWS S3 (Static hosting)
- AWS CloudFront (CDN)
- AWS API Gateway (Backend API)
- AWS Lambda (Serverless compute)
- AWS DynamoDB (NoSQL database)
- GitHub Actions (CI/CD)
- Terraform (Infrastructure as Code)
  
## Features
- Static resume website hosted on S3
- Serverless visitor counter using Lambda + DynamoDB
- API Gateway endpoint for frontend-backend communication
- CI/CD pipeline for automatic deployment
- Secure and scalable cloud architecture

## CICD Pipeline
 Code pushed to GitHub
- GitHub Actions triggers deployment
- Files synced to S3
- CloudFront cache invalidated

## Securitty
- IAM roles with least privilege
- HTTPS enabled via CloudFront
- Secure API communication

## What i leearnt
Building serverless applications on AWS
- Integrating frontend with backend APIs
- Automating deployments using CI/CD
- Managing infrastructure with Terraform

## Author 
David Digheji
Cloud Engineer | DevSecOps 
