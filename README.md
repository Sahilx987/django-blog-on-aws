# django-blog-on-aws
Production-grade Django blog application deployed on AWS EC2, RDS, S3, and CloudFront

## Architecture
-EC2 (App Server)
-RDS (MySQL)
- S3 (Static + Media)
- CloudFront (CDN)
- Route53 + ACM (SSL + DNS)
- IAM + VPC + CloudWatch

## Features
- Scalable and secure architecture
- HTTPS-enabled blog with custom domain
- Automated backups and monitoring

## How It Works
1. User accesses blog via CloudFront
2. Request goes to EC2 behind ALB
3. App reads/writes data to RDS
4. Media served via S3 + CloudFront
