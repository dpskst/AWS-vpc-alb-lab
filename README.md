# Project 06. AWS VPC 기반 Private EC2 + ALB 구축

## 1. Project Overview

AWS VPC 환경에서 Public / Private Subnet을 분리하고,

- Internet Gateway
- Route Table
- Security Group
- Public EC2
- Private EC2
- Bastion Host
- Application Load Balancer
- Target Group

을 직접 구성하여 AWS 네트워크 아키텍처를 구축했다.

특히 Public IP가 없는 Private EC2를 구성하고,
Public EC2를 Bastion Host로 사용하여 Private EC2에 SSH 접속하는 구조를 구현했다.

또한 Internet-facing ALB를 통해 Private EC2의 웹 서비스를 외부에서 접근할 수 있도록 구성하고 Target Health Check를 통해 정상 통신 여부를 검증했다.

---

## 2. Project Goals

- AWS VPC 네트워크 구조 이해
- Public / Private Subnet 구성
- Internet Gateway와 Route Table 구성
- EC2 네트워크 구성
- Security Group 기반 접근 제어
- Bastion Host를 통한 Private EC2 접근
- Application Load Balancer 구성
- Target Group 및 Health Check 구성
- Private EC2 서비스의 외부 접근 구조 구현
- AWS 네트워크 트러블슈팅 경험
- 향후 Terraform 기반 IaC 구축을 위한 AWS 인프라 이해

---

## 3. Architecture

```text
                         Internet
                            │
                            │ HTTP :80
                            ▼
                  ┌────────────────────┐
                  │ Application Load   │
                  │ Balancer           │
                  │ aws-devops-alb     │
                  └─────────┬──────────┘
                            │
                            │ HTTP :80
                            ▼
                 ┌──────────────────────┐
                 │ Private Subnet       │
                 │ 10.10.2.0/24         │
                 │                      │
                 │ Private EC2          │
                 │ 10.10.2.105          │
                 │ HTTP :80             │
                 └──────────────────────┘


Windows PC
    │
    │ SSH :22
    ▼
┌───────────────────────┐
│ Public EC2            │
│ Bastion Host          │
│ 10.10.1.x             │
└───────────┬───────────┘
            │
            │ SSH :22
            ▼
┌───────────────────────┐
│ Private EC2           │
│ 10.10.2.105           │
│ Public IP 없음        │
└───────────────────────┘
