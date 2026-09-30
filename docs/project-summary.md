# 프로젝트 요약

## 한 줄 요약

AWS VPC에서 Public/Private 네트워크를 분리하고 Bastion Host와 ALB를 이용해 Private EC2 서비스를 외부에 안전하게 노출하는 인프라를 구축했다.

## 면접/경력기술서용 요약

> AWS VPC(10.10.0.0/16)를 기반으로 Public/Private Subnet, Internet Gateway, Route Table, Security Group을 구성하고 Public EC2를 Bastion Host로 활용하여 Private EC2에 대한 제한적인 SSH 접근 구조를 구축했습니다. 이후 Internet-facing ALB와 Target Group을 구성하여 Public IP가 없는 Private EC2의 웹 서비스를 ALB를 통해 외부에서 접근할 수 있도록 구성하고 Health Check 및 Security Group 정책을 검증했습니다.

## 핵심 키워드

```text
AWS
VPC
Subnet
Route Table
Internet Gateway
EC2
Security Group
Bastion Host
Application Load Balancer
Target Group
Health Check
Private Subnet
Network Security
Linux
```
