AWS Production Smart eCommerce Platform
Complete Step-by-Step Deployment Roadmap — Amazon Linux
Purpose: This document is a practical deployment runbook. Follow the sections in order.

Important: Replace every value marked REPLACE_ME, YOUR_*, <...>, or example.com with your own value before running the command.

Operating System: Amazon Linux only. Ubuntu commands such as apt are intentionally not used.

Architecture: 8 subnets, CloudFront, Route 53, WAF, ACM, ALB, ASG, Launch Template + app-server AMI, Nginx, private frontend/backend, RDS, Prometheus, Grafana, PagerDuty, CloudWatch, Lambda, S3, SSM, NAT Gateway, VPC Endpoints, Secrets Manager, and a controlled public monitoring proxy.

ARCHITECTURE OF 3-TIRE  E-COMMERCE APPLICATION.

                            INTERNET
                               │
                               ▼
                            Route 53
                               │
                               ▼
                           CloudFront
                               │
                               ▼
                              WAF
                               │
                               ▼
                   Application Load Balancer
                           HTTPS :443
                               │
                +--------------+--------------+
                |                             |
                v                             v
       Frontend-Private-1             Frontend-Private-2
                |                             |
                +--------------+--------------+
                               |
                       Nginx Reverse Proxy
                               |
                +--------------+--------------+
                |                             |
                v                             v
         Backend-Private-1             Backend-Private-2
                |                             |
                +--------------+--------------+
                               |
                               v
                              RDS
                        DB Private-1 / 2


                MONITORING ACCESS
                       |
                   Developer
                       |
                       v
              Public Proxy Server
                       |
                     Nginx
                +------+------+
                |             |
                v             v
             Grafana       Prometheus
             private        private
                |
                v
            PagerDuty


                LOGGING
                   |
           CloudWatch / Sources
                   |
                Lambda
                   |
                   v
                  S3


              ADMINISTRATION
                   |
                  SSM
                   |
    +--------------+---------------+
    |              |               |
 Frontend       Backend        Monitoring
  Servers        Servers        Servers
