# Project 06. AWS VPC 기반 Private EC2 + ALB 구축

## 1. 프로젝트 개요

AWS VPC를 기반으로 Public Subnet과 Private Subnet을 분리하고, 각각 Public EC2와 Private EC2를 배치했습니다.

Public EC2는 Bastion Host로 활용하여 외부에서 Private EC2를 관리할 수 있도록 했으며, Internet-facing ALB를 통해 외부 HTTP 요청이 Private EC2의 Web Server로 전달되도록 구성했습니다.

NAT Gateway 없이 VPC, Subnet, Route Table, Internet Gateway, Security Group, Bastion Host, ALB, Target Group 간의 네트워크 흐름과 접근 제어 방식을 직접 확인했습니다.


---

## 2. 프로젝트 목표

이번 프로젝트의 주요 목표는 다음과 같습니다.

- AWS VPC 생성
- CIDR 구성
- Public Subnet 구성
- Private Subnet 구성
- Availability Zone 구성
- Internet Gateway 구성
- Route Table 구성
- Public EC2 구성
- Private EC2 구성
- Bastion Host 구성
- Private EC2 SSH 접속
- Security Group을 이용한 접근 제어
- Internet-facing ALB 구성
- Target Group 구성
- ALB Health Check 구성
- ALB → Private EC2 연결
- Private EC2의 인터넷 접근 제한 확인
- AWS VPC 네트워크 트래픽 흐름 이해


---

## 3. 전체 Architecture

Windows PC는 AWS VPC 외부의 사용자 환경입니다.

관리자 접속은 다음과 같은 경로로 구성했습니다.

Windows PC → Internet → Public EC2(Bastion Host) → Private EC2

사용자 서비스 접근은 다음과 같은 경로로 구성했습니다.

Internet → Internet-facing ALB → Target Group → Private EC2 → Web Server

~~~text
                                INTERNET
                          ┌────────┴────────┐
                          │                 │
                       HTTP :80          SSH :22
                          │                 │
                          ▼                 ▼
                   ┌────────────┐    ┌─────────────┐
                   │    ALB     │    │ Windows PC  │
                   │ Internet-  │    │  Local PC   │
                   │   facing   │    └──────┬──────┘
                   └─────┬──────┘           │
                         │                  │
                      HTTP :80            SSH :22
                         │                  │
                         │                  ▼
                         │          ┌───────────────┐
                         │          │  Public EC2   │
                         │          │ Bastion Host  │
                         │          └───────┬───────┘
                         │                  │
                         │               SSH :22
                         │                  │
                         ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AWS VPC                                 │
│                       10.10.0.0/16                              │
│                                                                 │
│  ┌─────────────────────────┐     ┌───────────────────────────┐  │
│  │ Public Subnet           │     │ Private Subnet            │  │
│  │ 10.10.1.0/24            │     │ 10.10.2.0/24              │  │
│  │                         │     │                           │  │
│  │ Public EC2              │────▶│ Private EC2              │  │
│  │ Bastion Host            │ SSH │ 10.10.2.105               │  │
│  │                         │     │ HTTP :80                  │  │
│  └─────────────────────────┘     └───────────────────────────┘  │
│                                                                 │
│  ALB는 Public Subnet 2개에 연결                                  │
│  - us-east-2a : aws-devops-public-subnet                        │
│  - us-east-2b : aws-devops-public-subnet-2                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
~~~

### 관리자 SSH 접근

~~~text
Windows PC
    │
    │ SSH :22
    ▼
Internet
    │
    ▼
Public EC2
(Bastion Host)
    │
    │ SSH :22
    ▼
Private EC2
10.10.2.105
~~~

### 사용자 HTTP 접근

~~~text
Internet
    │
    │ HTTP :80
    ▼
Internet-facing ALB
    │
    │ HTTP :80
    ▼
Target Group
    │
    ▼
Private EC2
    │
    ▼
Web Server
~~~

---

## 4. AWS 환경

### Region

~~~text
us-east-2 (Ohio)
~~~

### Availability Zone

~~~text
us-east-2a
us-east-2b
~~~

ALB는 서로 다른 Availability Zone의 Public Subnet에 연결했습니다.

### 주요 환경 정보

| 항목 | 구성 |
|---|---|
| AWS Region | us-east-2 |
| AZ | us-east-2a / us-east-2b |
| VPC | aws-devops-vpc |
| VPC CIDR | 10.10.0.0/16 |
| Public Subnet | 10.10.1.0/24 |
| Public Subnet 2 | 10.10.3.0/24 |
| Private Subnet | 10.10.2.0/24 |
| Public EC2 | aws-devops-vpc-ec2 |
| Private EC2 | aws-devops-private-ec2 |
| ALB | aws-devops-alb |
| Target Group | aws-devops-tg |

###  SCREENSHOT 02 - AWS Region

<img width="1541" height="335" alt="image" src="https://github.com/user-attachments/assets/49cc08d1-38cb-4d0a-b7b1-7fa02da66923" />



---

## 5. VPC 구성

### VPC 정보

~~~text
Name
aws-devops-vpc

CIDR
10.10.0.0/16
~~~

VPC의 전체 네트워크 범위는 다음과 같습니다.

~~~text
10.10.0.0 ~ 10.10.255.255
~~~

이 VPC 내부에 Public Subnet과 Private Subnet을 구성했습니다.

###  SCREENSHOT 03 - VPC

> <img width="1420" height="288" alt="image" src="https://github.com/user-attachments/assets/e0b620da-8d65-45e1-beb0-edf642c223de" />

>
> - VPC Name: aws-devops-vpc
> - IPv4 CIDR: 10.10.0.0/16


---

## 6. Subnet 구성

### 6.1 Public Subnet

~~~text
Name
aws-devops-public-subnet

CIDR
10.10.1.0/24

Availability Zone
us-east-2a
~~~

Public Subnet에는 Public EC2를 배치했습니다.


### 6.2 Public Subnet 2

~~~text
Name
aws-devops-public-subnet-2

CIDR
10.10.3.0/24

Availability Zone
us-east-2b
~~~

ALB가 서로 다른 Availability Zone에 연결될 수 있도록 두 번째 Public Subnet을 구성했습니다.


### 6.3 Private Subnet

~~~text
Name
aws-devops-private-subnet

CIDR
10.10.2.0/24

Availability Zone
us-east-2a
~~~

Private Subnet에는 Public IP가 없는 Private EC2를 배치했습니다.


### Subnet 구성 요약

| Subnet | CIDR | AZ | 용도 |
|---|---|---|---|
| aws-devops-public-subnet | 10.10.1.0/24 | us-east-2a | Public EC2 / ALB |
| aws-devops-public-subnet-2 | 10.10.3.0/24 | us-east-2b | ALB |
| aws-devops-private-subnet | 10.10.2.0/24 | us-east-2a | Private EC2 |

###  SCREENSHOT 04 - Subnet

> <img width="1287" height="209" alt="image" src="https://github.com/user-attachments/assets/00c68abd-e5d1-41c8-b9f4-2ff3b2cde384" />

>
> 가능하면 다음 항목이 함께 보이도록 합니다.
>
> - Name
> - CIDR
> - Availability Zone
> - VPC


---

## 7. Internet Gateway

### Internet Gateway 정보

~~~text
Name
aws-devops-igw

Attached VPC
aws-devops-vpc
~~~

Internet Gateway를 VPC에 연결하여 Public Subnet의 리소스가 인터넷과 통신할 수 있도록 구성했습니다.

###  SCREENSHOT 05 - Internet Gateway

> <img width="1095" height="187" alt="image" src="https://github.com/user-attachments/assets/7c09a776-34da-4614-baed-7619824d8f6f" />

>
> - aws-devops-igw
> - Attached VPC: aws-devops-vpc


---

## 8. Route Table 구성

### 8.1 Public Route Table

~~~text
Name
aws-devops-public-rt
~~~

Public Route Table에는 다음과 같은 기본 경로를 구성했습니다.

~~~text
Destination        Target

0.0.0.0/0          Internet Gateway
                   aws-devops-igw
~~~

Public Subnet은 Public Route Table에 연결했습니다.

~~~text
aws-devops-public-subnet
aws-devops-public-subnet-2
        │
        ▼
aws-devops-public-rt
        │
        ▼
aws-devops-igw
        │
        ▼
Internet
~~~


### 8.2 Private Route Table

~~~text
Name
aws-devops-private-rt
~~~

Private Subnet을 Private Route Table에 연결했습니다.

~~~text
aws-devops-private-subnet
        │
        ▼
aws-devops-private-rt
~~~

이번 프로젝트에서는 NAT Gateway를 사용하지 않았기 때문에 Private Route Table에 NAT Gateway를 통한 인터넷 경로를 구성하지 않았습니다.


### 📸 SCREENSHOT 06 - Public Route Table

> `aws-devops-public-rt` 화면에서 다음 내용을 확인할 수 있도록 캡처합니다.
>
> - 0.0.0.0/0
> - Internet Gateway
> - Subnet Association


### 📸 SCREENSHOT 07 - Private Route Table

> `aws-devops-private-rt` 화면에서 다음 내용을 확인할 수 있도록 캡처합니다.
>
> - Routes
> - Subnet Association
>
> Private Route Table에 NAT Gateway가 없는 것도 확인할 수 있습니다.


---

## 9. Public EC2 구성

### EC2 정보

~~~text
Name
aws-devops-vpc-ec2

Role
Bastion Host / Jump Host

Instance Type
t3.micro

Subnet
aws-devops-public-subnet

Availability Zone
us-east-2a

Public IP
3.143.203.86

Security Group
aws-devops-vpc-sg
~~~

Public EC2에는 Public IP가 할당되어 있어 Windows PC에서 SSH 접속할 수 있습니다.


### Public EC2의 역할

Public EC2는 Private EC2에 접근하기 위한 Bastion Host 역할을 수행합니다.

~~~text
Windows PC
    │
    │ SSH
    ▼
Public EC2
    │
    │ SSH
    ▼
Private EC2
~~~

이를 통해 Private EC2에 Public IP를 직접 할당하지 않고도 관리할 수 있습니다.


###  SCREENSHOT 08 - Public EC2

> <img width="1263" height="528" alt="image" src="https://github.com/user-attachments/assets/df6fe686-f34b-4297-9ed8-1c5ac8a5c687" />

>
> - Instance Type
> - Public IPv4
> - Private IPv4
> - Subnet
> - Availability Zone
> - Security Group


---

## 10. Private EC2 구성

### EC2 정보

~~~text
Name
aws-devops-private-ec2

Instance Type
t3.micro

Operating System
Amazon Linux 2023

Subnet
aws-devops-private-subnet

Private IP
10.10.2.105

Public IP
없음

Security Group
aws-devops-private-sg
~~~

Private EC2에는 Public IP를 할당하지 않았습니다.

따라서 인터넷에서 Private EC2로 직접 SSH 접속할 수 없습니다.

###  SCREENSHOT 09 - Private EC2

> <img width="1228" height="524" alt="image" src="https://github.com/user-attachments/assets/0c8779ec-9d57-4e43-ad1e-9437948e1b70" />

>
> - Private IPv4: 10.10.2.105
> - Public IPv4가 없는 상태
> - Subnet
> - Security Group
> - Instance Type


---

## 11. Private EC2 Web Server 구성

Private Subnet에서는 NAT Gateway를 사용하지 않았기 때문에 인터넷을 통해 Nginx 패키지를 설치하지 않았습니다.

대신 Python의 기본 HTTP Server를 사용하여 Web Server를 구성했습니다.

### 11.1 Web Server 파일 생성

~~~bash
echo "AWS DevOps Private EC2 - ALB TEST" | sudo tee /tmp/index.html
~~~

### 11.2 Python HTTP Server 실행

~~~bash
sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
~~~

### 11.3 로컬 접속 테스트

~~~bash
curl http://localhost
~~~

결과:

~~~text
AWS DevOps Private EC2 - ALB TEST
~~~

Private EC2에서 Web Server가 정상적으로 동작하는 것을 확인했습니다.


###  SCREENSHOT 10 - Private EC2 Web Server

> <img width="409" height="41" alt="image" src="https://github.com/user-attachments/assets/d0775d3a-c151-4c07-be66-036261936dde" />

>
> `curl http://localhost`
>
> 결과:
>
> `AWS DevOps Private EC2 - ALB TEST`


---

## 12. Bastion Host를 이용한 Private EC2 접속

Windows PC에서 Public EC2를 거쳐 Private EC2에 접속했습니다.

Windows PowerShell에서 다음 명령어를 사용했습니다.

~~~powershell
ssh -o "ProxyCommand=ssh -i `"$HOME\Downloads\aws-devops-lab-key.pem`" -W %h:%p ec2-user@3.143.203.86" -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@10.10.2.105
~~~

정상적으로 접속하면 다음과 같이 Private EC2 Shell을 확인할 수 있습니다.

~~~text
[ec2-user@ip-10-10-2-105 ~]$
~~~

접속 구조:

~~~text
Windows PC
     │
     │ SSH
     ▼
Public EC2
3.143.203.86
     │
     │ SSH
     ▼
Private EC2
10.10.2.105
~~~

###  SCREENSHOT 11 - Bastion SSH 접속

> <img width="958" height="273" alt="image" src="https://github.com/user-attachments/assets/51c20d4b-1528-4b0b-862f-ba1abcdd4539" />
>
> - PowerShell
> - SSH 명령어
> - `ip-10-10-2-105`
> - Private EC2 Shell


---

## 13. Security Group 구성

이번 프로젝트에서는 Security Group을 이용하여 각 구성 요소 간 접근을 제한했습니다.


### 13.1 Public EC2 Security Group

~~~text
Name
aws-devops-vpc-sg
~~~

Public EC2는 Windows PC에서 SSH 접속할 수 있도록 SSH 22번 포트를 허용했습니다.


### 13.2 Private EC2 Security Group

~~~text
Name
aws-devops-private-sg
~~~

Private EC2에는 다음과 같이 접근을 제한했습니다.

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | aws-devops-vpc-sg |
| HTTP | 80 | aws-devops-alb-sg |

Private EC2의 SSH는 Public EC2의 Security Group에서 들어오는 요청만 허용했습니다.

Private EC2의 HTTP는 ALB Security Group에서 들어오는 요청만 허용했습니다.


### 13.3 ALB Security Group

~~~text
Name
aws-devops-alb-sg
~~~

ALB는 인터넷에서 HTTP 요청을 받을 수 있도록 구성했습니다.

| Type | Port | Source |
|---|---:|---|
| HTTP | 80 | 0.0.0.0/0 |


### 최종 접근 구조

~~~text
Internet
    │
    │ HTTP :80
    ▼
ALB
    │
    │ HTTP :80
    ▼
Private EC2


Windows PC
    │
    │ SSH :22
    ▼
Public EC2
    │
    │ SSH :22
    ▼
Private EC2
~~~

---

## 14. Application Load Balancer 구성

### ALB 정보

~~~text
Name
aws-devops-alb

Scheme
Internet-facing

IP address type
IPv4
~~~

ALB는 다음 두 Public Subnet에 연결했습니다.

~~~text
us-east-2a
aws-devops-public-subnet

us-east-2b
aws-devops-public-subnet-2
~~~

ALB는 두 개의 Availability Zone에 걸쳐 구성했습니다.


###  SCREENSHOT 13 - ALB 구성

> <img width="1368" height="385" alt="image" src="https://github.com/user-attachments/assets/89a27413-43d4-4bf1-8226-98a0e6d0f084" />
>
> - Internet-facing
> - IPv4
> - Availability Zone
> - Security Group
> - DNS Name


---

## 15. Target Group 구성

### Target Group 정보

~~~text
Name
aws-devops-tg

Target Type
Instances

Protocol
HTTP

Port
80
~~~

Target으로 Private EC2를 등록했습니다.

~~~text
Private EC2
10.10.2.105
Port 80
~~~


### Health Check

Health Check는 다음과 같이 구성했습니다.

~~~text
Protocol
HTTP

Port
Traffic Port

Path
/
~~~

처음에는 Private EC2의 HTTP 80 포트가 ALB Security Group으로부터 허용되지 않아 Health Check가 `Unhealthy` 상태였습니다.

Private EC2 Security Group에 다음 규칙을 추가했습니다.

~~~text
HTTP 80
Source:
aws-devops-alb-sg
~~~

이후 Target Group의 Health Status가 다음과 같이 변경되었습니다.

~~~text
Healthy
~~~

###  SCREENSHOT 14 - Target Group Healthy

> <img width="1388" height="436" alt="image" src="https://github.com/user-attachments/assets/14229552-e801-4816-865d-fcdf986da0db" />

>
> - Private EC2
> - Port 80
> - Health status: Healthy
>
> ALB → Target Group → Private EC2 연결이 정상이라는 것을 보여주는 핵심 자료입니다.


---

## 16. ALB → Private EC2 테스트

ALB의 DNS 주소를 브라우저에서 접속하여 테스트했습니다.

~~~text
Internet
   │
   │ HTTP :80
   ▼
Internet-facing ALB
   │
   │ HTTP :80
   ▼
Target Group
   │
   ▼
Private EC2
   │
   ▼
Python HTTP Server
~~~

브라우저에서 다음 응답을 확인했습니다.

~~~text
AWS DevOps Private EC2 - ALB TEST
~~~

이를 통해 다음 구조가 정상적으로 동작하는 것을 확인했습니다.

~~~text
Internet
   ↓
Internet-facing ALB
   ↓
Target Group
   ↓
Private EC2
   ↓
Web Server
~~~

###  SCREENSHOT 15 - ALB 접속 결과

<img width="882" height="186" alt="image" src="https://github.com/user-attachments/assets/89206576-2a74-466e-8e91-fdd98079af0e" />


---

## 17. Private EC2 인터넷 접근 테스트

Private EC2에서 외부 인터넷 접근 여부를 확인했습니다.

~~~bash
curl -I https://www.google.com
~~~

Private EC2에는 NAT Gateway가 구성되어 있지 않기 때문에 외부 인터넷으로 직접 통신할 수 없는 것을 확인했습니다.

이를 통해 Public Subnet과 Private Subnet의 네트워크 차이를 확인했습니다.

###  SCREENSHOT 16 - Private EC2 인터넷 접근 제한

<img width="519" height="86" alt="image" src="https://github.com/user-attachments/assets/dec5a595-be38-4d2e-97af-8641aae46228" />


---

## 20. Troubleshooting

### 20.1 Private EC2에서 dnf install이 되지 않는 문제

Private EC2에서 다음 명령어를 실행했을 때 정상적으로 패키지를 다운로드하지 못했습니다.

~~~bash
sudo dnf install nginx -y
~~~

원인은 Private Subnet에서 인터넷으로 나가는 NAT Gateway가 구성되어 있지 않기 때문입니다.

이번 프로젝트에서는 무료 실습을 위해 NAT Gateway를 사용하지 않았습니다.

대신 Python HTTP Server를 사용하여 Web Server를 구성했습니다.

~~~bash
sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
~~~


### 20.2 ALB Health Check가 Unhealthy인 문제

처음 ALB Target Group의 Health Check가 다음 상태였습니다.

~~~text
Unhealthy
~~~

원인은 Private EC2 Security Group에서 ALB의 HTTP 요청을 허용하지 않았기 때문입니다.

Private EC2 Security Group에 다음 규칙을 추가했습니다.

~~~text
HTTP
Port: 80
Source: aws-devops-alb-sg
~~~

이후 다음과 같이 변경되었습니다.

~~~text
Healthy
~~~


### 20.3 Private EC2에 직접 SSH 접속이 되지 않는 문제

Private EC2에는 Public IP가 없기 때문에 Windows PC에서 직접 SSH 접속할 수 없습니다.

잘못된 접근:

~~~text
Windows PC
    │
    │ SSH
    ▼
Private EC2
~~~

올바른 접근:

~~~text
Windows PC
    │
    ▼
Public EC2
(Bastion Host)
    │
    ▼
Private EC2
~~~


### 20.4 Security Group 수정 오류

기존에 `0.0.0.0/0`으로 등록된 IPv4 규칙을 Security Group Source로 변경하려고 했을 때 다음과 같은 오류가 발생했습니다.

~~~text
existing IPv4 CIDR rule cannot specify referenced group ID
~~~

기존 CIDR 규칙을 삭제한 후 Security Group을 Source로 사용하는 새로운 규칙을 생성하여 해결했습니다.


---

## 21. 보안 구성

이번 프로젝트에서는 Security Group을 이용하여 접근 범위를 분리했습니다.

### 서비스 접근

~~~text
Internet
   │
   │ HTTP :80
   ▼
ALB
   │
   │ HTTP :80
   ▼
Private EC2
~~~

ALB에서 Private EC2로 들어오는 HTTP 트래픽만 허용했습니다.


### 관리자 접근

~~~text
Windows PC
   │
   │ SSH :22
   ▼
Public EC2
   │
   │ SSH :22
   ▼
Private EC2
~~~

Private EC2의 SSH 포트는 인터넷 전체에 공개하지 않고 Public EC2의 Security Group을 Source로 지정했습니다.


---

## 24. 프로젝트를 통해 학습한 내용

### AWS Network

- VPC
- CIDR
- Public Subnet
- Private Subnet
- Availability Zone
- Internet Gateway
- Route Table

### AWS Compute

- EC2
- Public EC2
- Private EC2
- Bastion Host

### AWS Security

- Security Group
- Port 기반 접근 제어
- Security Group Reference
- Public / Private 네트워크 분리

### AWS Load Balancing

- Application Load Balancer
- Internet-facing ALB
- Target Group
- Health Check
- ALB → Private EC2 연결

### Linux

- SSH
- ProxyCommand
- ip route
- ip addr
- curl
- Python HTTP Server
- nohup


---

## 25. 프로젝트 핵심 정리

이번 프로젝트의 핵심은 단순히 EC2를 생성하는 것이 아니라 AWS의 기본적인 네트워크 구조를 직접 구축하고 각 구성 요소의 역할을 확인한 것입니다.

~~~text
VPC
 ↓
Subnet
 ↓
Route Table
 ↓
Internet Gateway
 ↓
Security Group
 ↓
Public / Private EC2
 ↓
Bastion Host
 ↓
Application Load Balancer
 ↓
Target Group
 ↓
Private EC2 Web Server
~~~

특히 Public EC2와 Private EC2를 분리하고 Bastion Host를 통해 Private EC2에 접근하는 구조를 직접 구현했습니다.

또한 Internet-facing ALB를 통해 인터넷에서 Private EC2의 Web Server까지 HTTP 요청이 전달되는 구조를 구축하고 Health Check를 통해 Target 상태를 확인했습니다.


---

## 26. 향후 확장 계획

이번 프로젝트를 기반으로 다음 단계의 DevOps 프로젝트로 확장할 예정입니다.

~~~text
Project 06
AWS VPC + EC2 + ALB
        │
        ▼
Project 07
Terraform
        │
        ▼
AWS Infrastructure as Code
        │
        ▼
Project 08
ECR + Docker
        │
        ▼
Project 09
GitHub Actions CI/CD
        │
        ▼
Project 10
ECS / EKS
        │
        ▼
Project 11
Prometheus + Grafana
        │
        ▼
Project 12
AWS DevSecOps
~~~

기존 Linux / Server / Security 운영 경험을 기반으로 AWS Cloud, Infrastructure as Code, Container, CI/CD, Kubernetes 및 DevSecOps 영역까지 역량을 확장하는 것을 목표로 합니다.


---

## 28. 프로젝트 한 줄 요약

> AWS VPC 환경에서 Public / Private Subnet을 분리하고 Bastion Host를 통한 Private EC2 관리 및 Internet-facing ALB를 통한 Private Web Server 서비스를 구현했습니다.
