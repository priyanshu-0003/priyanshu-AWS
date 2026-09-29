AWS Production Smart eCommerce Platform
Complete Step-by-Step Deployment Roadmap — Amazon Linux
Purpose: This document is a practical deployment runbook. Follow the sections in order.

Important: Replace every value marked REPLACE_ME, YOUR_*, <...>, or example.com with your own value before running the command.

Operating System: Amazon Linux only. Ubuntu commands such as apt are intentionally not used.

Architecture: 8 subnets, CloudFront, Route 53, WAF, ACM, ALB, ASG, Launch Template + app-server AMI, Nginx, private frontend/backend, RDS, Prometheus, Grafana, PagerDuty, CloudWatch, Lambda, S3, SSM, NAT Gateway, VPC Endpoints, Secrets Manager, and a controlled public monitoring proxy.

