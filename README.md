# AWS VPC + ALB 기반 Private EC2 서비스 구축

## 1. 프로젝트 개요

AWS VPC 환경에서 Public/Private Subnet을 직접 구성하고, Public EC2를 Bastion(점프 호스트)으로 사용하여 Private EC2에 SSH 접속하는 네트워크를 구축했다.

또한 Internet-facing Application Load Balancer(ALB)를 구성하여 **Public IP가 없는 Private EC2의 웹 서비스를 ALB를 통해 외부에서 접근**할 수 있도록 구성했다.

> 본 프로젝트는 AWS Console을 이용한 수동 구축 실습이며, 이후 Terraform(IaC)으로 동일한 인프라를 코드화하는 것을 확장 목표로 한다.

---

## 2. 구축 목표

- AWS VPC 및 CIDR 설계
- Public / Private Subnet 분리
- Internet Gateway 및 Route Table 구성
- Public EC2 구축
- Private EC2 구축
- Bastion Host를 통한 Private EC2 SSH 접근
- Security Group 기반 접근 제어
- Application Load Balancer 구성
- Target Group 및 Health Check 구성
- Private EC2 웹 서비스의 ALB 경유 외부 접근 확인
- Private Subnet의 직접 인터넷 접근 차단 구조 확인

---

## 3. 아키텍처

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
                ┌───────────────────────┐
                │ Private Subnet        │
                │ 10.10.2.0/24          │
                │                       │
                │ Private EC2           │
                │ 10.10.2.105           │
                │ HTTP :80              │
                └───────────────────────┘


Windows PC
    │
    │ SSH :22
    ▼
┌───────────────────────┐
│ Public EC2            │
│ Public IP             │
│ 10.10.1.x             │
│ Bastion / Jump Host   │
└───────────┬───────────┘
            │
            │ SSH :22
            ▼
     Private EC2
     10.10.2.105
```

### Subnet 구성

| 구분 | CIDR | 용도 |
|---|---|---|
| VPC | 10.10.0.0/16 | 전체 네트워크 |
| Public Subnet | 10.10.1.0/24 | Public EC2 / ALB |
| Private Subnet | 10.10.2.0/24 | Private EC2 |
| Public Subnet 2 | 10.10.3.0/24 | ALB의 두 번째 AZ 구성 |

### AWS 리전

- Region: `us-east-2` (Ohio)
- Public/Private 실습 리소스는 해당 리전에 구성

---

## 4. 주요 AWS 리소스

| 리소스 | 이름 | 역할 |
|---|---|---|
| VPC | `aws-devops-vpc` | 전체 네트워크 |
| Internet Gateway | `aws-devops-igw` | Public Subnet 인터넷 연결 |
| Public Route Table | `aws-devops-public-rt` | Public Subnet 라우팅 |
| Private Route Table | `aws-devops-private-rt` | Private Subnet 라우팅 |
| Public Subnet | `aws-devops-public-subnet` | Public EC2 |
| Private Subnet | `aws-devops-private-subnet` | Private EC2 |
| Public Subnet 2 | `aws-devops-public-subnet-2` | ALB용 두 번째 AZ |
| Public EC2 | `aws-devops-vpc-ec2` | Bastion / Jump Host |
| Private EC2 | `aws-devops-private-ec2` | 내부 웹 서버 |
| Private SG | `aws-devops-private-sg` | Private EC2 접근 제어 |
| Public SG | `aws-devops-vpc-sg` | Public EC2 접근 제어 |
| ALB SG | `aws-devops-alb-sg` | ALB 접근 제어 |
| Target Group | `aws-devops-tg` | Private EC2 대상 관리 |
| ALB | `aws-devops-alb` | 외부 HTTP 요청 수신 |

---

## 5. 네트워크 설계

### Public Subnet

```text
10.10.1.0/24
       │
       ▼
aws-devops-public-rt
       │
       └── 0.0.0.0/0 → Internet Gateway
```

Public EC2는 Public IPv4를 사용하여 인터넷에서 SSH 접속이 가능하도록 구성했다.

### Private Subnet

```text
10.10.2.0/24
       │
       ▼
aws-devops-private-rt
       │
       └── 인터넷으로 직접 연결되는 NAT Gateway 없음
```

Private EC2에는 Public IPv4를 할당하지 않았다.

따라서 Private EC2는 인터넷에서 직접 SSH 접속할 수 없고, Bastion Host를 통해 관리하도록 구성했다.

---

## 6. Security Group 설계

### Public EC2

```text
SSH :22
Source: 관리용 접근 허용 범위
```

### Private EC2

```text
SSH :22
Source: Public EC2의 Security Group

HTTP :80
Source: ALB의 Security Group
```

핵심은 Private EC2에 인터넷 전체(`0.0.0.0/0`)에서 SSH를 허용하지 않는 것이다.

```text
Internet
   X
   │
   └── Private EC2 :22 직접 접근 차단

Public EC2 SG
   │
   └── SSH :22
          ↓
     Private EC2
```

---

## 7. Bastion Host SSH 테스트

Windows PowerShell에서 Public EC2를 경유하여 Private EC2에 접속했다.

```powershell
ssh -o "ProxyCommand=ssh -i `"$HOME\Downloads\aws-devops-lab-key.pem`" -W %h:%p ec2-user@3.143.203.86" -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@10.10.2.105
```

접속 성공 시:

```text
[ec2-user@ip-10-10-2-105 ~]$
```

Private EC2의 주소가 표시된다.

### 확인 명령

```bash
hostname
ip -4 addr
ip route
```

실제 구축 환경에서 확인한 Private EC2 주소:

```text
10.10.2.105
```

---

## 8. Private EC2 네트워크 검증

Private EC2에서:

```bash
ip route
```

확인 결과 예:

```text
default via 10.10.2.1 dev ens5
10.10.0.2 via 10.10.2.1 dev ens5
10.10.2.0/24 dev ens5
10.10.2.1 dev ens5
```

Private EC2에서 외부 인터넷 접근을 시도했을 때 현재 구성에는 NAT Gateway가 없으므로 인터넷 접근이 불가능한 것을 확인했다.

```bash
curl -I https://www.google.com
```

> NAT Gateway는 비용이 발생할 수 있으므로 이번 무료 위주 실습에서는 구성하지 않았다.

---

## 9. ALB + Target Group

### Target Group

```text
Name: aws-devops-tg
Protocol: HTTP
Port: 80
Target: aws-devops-private-ec2
```

Health Check:

```text
Protocol: HTTP
Path: /
Port: traffic-port
```

처음에는 `Unhealthy` 상태가 발생했으며, Private EC2의 Security Group에서 ALB Security Group으로부터 HTTP/80 접근을 허용한 후 `Healthy` 상태로 변경되는 것을 확인했다.

---

## 10. Private EC2 웹 서비스

Private EC2는 인터넷에서 패키지를 직접 다운로드할 수 없는 구조이므로, 실습에서는 추가 패키지 설치 없이 Python 내장 HTTP 서버를 사용했다.

테스트 페이지:

```bash
echo "AWS DevOps Private EC2 - ALB TEST" | sudo tee /tmp/index.html
```

웹 서버 실행:

```bash
sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
```

로컬 테스트:

```bash
curl http://localhost
```

정상 응답:

```text
AWS DevOps Private EC2 - ALB TEST
```

---

## 11. 최종 검증

ALB의 DNS Name으로 접속:

```text
http://<ALB-DNS-NAME>
```

브라우저에서 다음 응답을 확인했다.

```text
AWS DevOps Private EC2 - ALB TEST
```

이를 통해 다음 경로를 검증했다.

```text
Internet
   ↓
Application Load Balancer :80
   ↓
Target Group
   ↓
Private EC2 10.10.2.105 :80
   ↓
Web Service
```

---

## 12. 트러블슈팅

### 문제 1. Private EC2에서 `dnf install nginx -y`가 오래 걸림

원인:

- Private Subnet에 NAT Gateway가 없음
- Private EC2가 인터넷으로 패키지 저장소에 접근할 수 없음

대응:

- NAT Gateway를 추가하지 않고 무료 실습을 위해 Python 내장 HTTP 서버 사용

---

### 문제 2. Target Group이 `Unhealthy`

원인:

- ALB가 Private EC2의 TCP/80에 접근할 수 있도록 Security Group이 구성되지 않음

확인:

```text
aws-devops-alb-sg
        │
        │ HTTP :80
        ▼
aws-devops-private-sg
```

Private EC2 Security Group에:

```text
HTTP
TCP 80
Source: aws-devops-alb-sg
```

를 추가한 후 Health Check가 `Healthy`로 변경됨.

---

### 문제 3. Private EC2 SSH 접근

Private EC2에는 Public IPv4가 없기 때문에 Windows에서 직접 SSH할 수 없다.

Bastion Host를 이용한다.

```text
Windows
   ↓
Public EC2
   ↓
Private EC2
```

---

## 13. 학습한 핵심 개념

### VPC

AWS에서 논리적으로 분리된 가상 네트워크 환경을 구성한다.

### Subnet

VPC의 IP 대역을 목적에 따라 분리한다.

### Internet Gateway

Public Subnet의 리소스가 인터넷과 통신할 수 있도록 연결한다.

### Route Table

패킷이 어느 대상으로 전달될지 결정한다.

### Security Group

EC2 및 ALB에 대한 상태 저장형 네트워크 접근 제어를 수행한다.

### Bastion Host

Private 네트워크의 서버에 관리자가 접근하기 위한 중간 접속 서버다.

### ALB

외부 HTTP 요청을 받아 Target Group에 등록된 애플리케이션 서버로 요청을 전달한다.

### Target Group

ALB가 요청을 전달할 대상과 Health Check를 관리한다.

---

## 14. 향후 확장 계획

```text
현재
AWS Console
VPC
 ├─ Public Subnet
 ├─ Private Subnet
 ├─ Bastion EC2
 ├─ Private EC2
 └─ ALB
       ↓
      완료

다음
Terraform
       ↓
IaC 기반 AWS 인프라 자동화
       ↓
Docker
       ↓
ECR
       ↓
GitHub Actions
       ↓
ECS / EKS
       ↓
Prometheus / Grafana
```

---

## 15. 비용 관리

이번 실습은 무료 또는 최소 비용을 목표로 구성했다.

특히 다음 리소스는 사용하지 않았다.

- NAT Gateway
- RDS
- 불필요한 추가 EC2
- 고사양 인스턴스

실습 종료 후에는 사용하지 않는 AWS 리소스를 확인하고 삭제한다.

> AWS의 무료 사용 혜택 및 과금 조건은 계정/시점에 따라 다를 수 있으므로 AWS Billing에서 실제 사용량과 비용을 확인한다.
