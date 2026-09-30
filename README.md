# Project 06. AWS VPC 기반 Private EC2 + ALB 구축

## 1. 프로젝트 개요

AWS 환경에서 VPC 네트워크를 직접 구성하고 Public Subnet과 Private Subnet을 분리하여 EC2 인스턴스를 배치했습니다.

Public Subnet에는 외부에서 접근할 수 있는 EC2를 구성하고, 해당 EC2를 Bastion Host(Jump Host)로 사용하여 Private Subnet의 EC2에 SSH 접속할 수 있도록 구성했습니다.

또한 Internet-facing ALB(Application Load Balancer)를 구성하여 외부 사용자가 Private EC2에 직접 접근하지 않고 ALB를 통해 Private EC2의 Web Server에 접근할 수 있도록 구성했습니다.

이번 프로젝트에서는 NAT Gateway를 사용하지 않고 AWS 네트워크 구조와 Public / Private Subnet의 차이, Route Table, Security Group, Bastion Host, ALB의 동작 방식을 직접 확인하는 것을 목표로 했습니다.


---

# 2. 프로젝트 목표

이번 프로젝트의 주요 목표는 다음과 같습니다.

- AWS VPC 생성
- Public / Private Subnet 구성
- Internet Gateway 구성
- Route Table 구성
- Public EC2 구성
- Private EC2 구성
- Bastion Host를 통한 Private EC2 SSH 접속
- Security Group을 이용한 접근 제어
- Internet-facing ALB 구성
- Target Group 구성
- ALB → Private EC2 연결
- ALB Health Check 확인
- Private EC2의 외부 인터넷 접근 제한 확인
- AWS VPC 네트워크 트래픽 흐름 이해


---

# 3. 전체 Architecture

Windows PC는 AWS VPC 외부의 사용자 환경입니다.

Windows PC에서 Internet을 통해 Public EC2에 SSH 접속하고, Public EC2를 Bastion Host로 사용하여 Private EC2에 접근합니다.

서비스 접근은 별도의 경로로 구성되어 있으며, 외부 사용자가 Internet-facing ALB로 HTTP 요청을 보내면 ALB가 Private EC2로 요청을 전달합니다.

```text
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
│                         AWS VPC                                │
│                       10.10.0.0/16                              │
│                                                                 │
│  ┌─────────────────────────────┐    ┌────────────────────────┐ │
│  │ Public Subnet               │    │ Private Subnet         │ │
│  │ 10.10.1.0/24                │    │ 10.10.2.0/24           │ │
│  │                             │    │                        │ │
│  │ Public EC2                  │    │ Private EC2             │ │
│  │ Bastion / Jump Host         │───▶│ 10.10.2.105             │ │
│  │                             │SSH │ HTTP :80                │ │
│  └─────────────────────────────┘    └────────────────────────┘ │
│                                                                 │
│  ALB는 Public Subnet 2개에 연결                                   │
│  - us-east-2a : aws-devops-public-subnet                        │
│  - us-east-2b : aws-devops-public-subnet-2                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
주요 트래픽 흐름
서비스 접근
Internet
   │
   │ HTTP :80
   ▼
Internet-facing ALB
   │
   │ HTTP :80
   ▼
Private EC2
   │
   ▼
Web Server
관리자 SSH 접근
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
📸 SCREENSHOT 01 - 전체 Architecture

AWS VPC / Subnet / ALB / EC2 구조를 정리한 최종 Architecture 이미지를 넣는 위치입니다.

추천 이미지:

draw.io 또는 PowerPoint로 직접 만든 Architecture

AWS 공식 Architecture 아이콘 활용

Windows PC는 VPC 외부에 배치

ALB / Public EC2 / Private EC2는 VPC 내부에 배치

4. AWS 환경
4.1 Region
Region
us-east-2 (Ohio)
4.2 Availability Zone
us-east-2a
us-east-2b
ALB는 서로 다른 Availability Zone의 Public Subnet에 연결했습니다.

📸 SCREENSHOT 02 - AWS Region / Resource 화면

EC2 또는 VPC 콘솔에서 프로젝트 리소스가 us-east-2에 생성되어 있는 화면을 캡처합니다.

5. VPC 구성
5.1 VPC 생성
VPC Name
aws-devops-vpc

CIDR
10.10.0.0/16
VPC의 전체 네트워크 범위는 다음과 같습니다.

10.10.0.0 ~ 10.10.255.255
이를 기준으로 Public Subnet과 Private Subnet을 분리했습니다.

📸 SCREENSHOT 03 - VPC 생성 화면

AWS Console → VPC → Your VPCs에서 aws-devops-vpc와 10.10.0.0/16이 보이는 화면을 캡처합니다.

6. Subnet 구성
6.1 Public Subnet
Name
aws-devops-public-subnet

CIDR
10.10.1.0/24

Availability Zone
us-east-2a
Public Subnet에는 외부에서 접근 가능한 Public EC2를 배치했습니다.

6.2 Public Subnet 2
Name
aws-devops-public-subnet-2

CIDR
10.10.3.0/24

Availability Zone
us-east-2b
ALB가 서로 다른 Availability Zone에 연결될 수 있도록 두 번째 Public Subnet을 구성했습니다.

6.3 Private Subnet
Name
aws-devops-private-subnet

CIDR
10.10.2.0/24

Availability Zone
us-east-2a
Private Subnet에는 Public IP가 없는 Private EC2를 배치했습니다.

📸 SCREENSHOT 04 - Subnet 구성 화면

AWS Console → VPC → Subnets에서 다음 3개의 Subnet이 보이는 화면을 캡처합니다.

aws-devops-public-subnet

aws-devops-public-subnet-2

aws-devops-private-subnet

가능하면 Name / CIDR / Availability Zone이 함께 보이도록 캡처합니다.

7. Internet Gateway
7.1 Internet Gateway 생성
Name
aws-devops-igw
생성한 Internet Gateway를 다음 VPC에 연결했습니다.

aws-devops-vpc
Internet Gateway는 VPC와 인터넷 사이의 통신을 가능하게 합니다.

📸 SCREENSHOT 05 - Internet Gateway

AWS Console → VPC → Internet Gateways에서 aws-devops-igw가 aws-devops-vpc에 Attached 된 화면을 캡처합니다.

8. Route Table 구성
8.1 Public Route Table
Name
aws-devops-public-rt
다음 Route를 추가했습니다.

Destination        Target
0.0.0.0/0          Internet Gateway
                   aws-devops-igw
Public Subnet을 Public Route Table에 연결했습니다.

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
8.2 Private Route Table
Name
aws-devops-private-rt
Private Subnet을 Private Route Table에 연결했습니다.

aws-devops-private-subnet
        │
        ▼
aws-devops-private-rt
이번 프로젝트에서는 NAT Gateway를 사용하지 않았기 때문에 Private Route Table에 NAT Gateway를 통한 인터넷 경로를 구성하지 않았습니다.

📸 SCREENSHOT 06 - Public Route Table

AWS Console → VPC → Route Tables → aws-devops-public-rt

다음 내용이 함께 보이도록 캡처합니다.

0.0.0.0/0

Internet Gateway

Subnet Association

📸 SCREENSHOT 07 - Private Route Table

aws-devops-private-rt의 Routes와 Subnet Association을 캡처합니다.

9. Public EC2 구성
9.1 EC2 정보
Name
aws-devops-vpc-ec2

Role
Bastion Host / Jump Host

Subnet
aws-devops-public-subnet

Availability Zone
us-east-2a

Security Group
aws-devops-vpc-sg

Public IP
3.143.203.86
Public EC2에는 Public IP가 할당되어 있어 Windows PC에서 SSH 접속이 가능합니다.

9.2 Public EC2의 역할
Public EC2는 Private EC2에 접근하기 위한 Bastion Host 역할을 수행합니다.

Windows PC
    │
    │ SSH
    ▼
Public EC2
    │
    │ SSH
    ▼
Private EC2
이를 통해 Private EC2에 Public IP를 직접 할당하지 않고도 관리할 수 있습니다.

📸 SCREENSHOT 08 - Public EC2

EC2 Console에서 aws-devops-vpc-ec2의 다음 정보가 보이도록 캡처합니다.

Instance ID

Instance Type

Public IPv4

Private IPv4

Subnet

Security Group

10. Private EC2 구성
10.1 EC2 정보
Name
aws-devops-private-ec2

Instance Type
t3.micro

Subnet
aws-devops-private-subnet

Private IP
10.10.2.105

Public IP
없음

Operating System
Amazon Linux 2023

Security Group
aws-devops-private-sg
Private EC2에는 Public IP를 할당하지 않았습니다.

따라서 인터넷에서 Private EC2로 직접 접근할 수 없습니다.

📸 SCREENSHOT 09 - Private EC2

EC2 Console에서 aws-devops-private-ec2를 선택한 후 다음 내용이 보이도록 캡처합니다.

Private IPv4

Public IPv4가 없는 상태

Subnet

Security Group

Instance Type

11. Private EC2 Web Server 구성
Private Subnet에서는 NAT Gateway를 사용하지 않았기 때문에 인터넷을 통해 Nginx 패키지를 설치하지 않았습니다.

대신 Python의 기본 HTTP Server를 사용하여 Web Server를 구성했습니다.

11.1 Web Server 파일 생성
echo "AWS DevOps Private EC2 - ALB TEST" | sudo tee /tmp/index.html
11.2 Python HTTP Server 실행
sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
11.3 로컬 접속 테스트
curl http://localhost
결과:

AWS DevOps Private EC2 - ALB TEST
Private EC2에서 Web Server가 정상적으로 동작하는 것을 확인했습니다.

📸 SCREENSHOT 10 - Private EC2 Web Server

SSH로 Private EC2에 접속한 후 다음 명령어와 결과가 함께 보이도록 캡처합니다.

curl http://localhost
결과:

AWS DevOps Private EC2 - ALB TEST

12. Bastion Host를 이용한 Private EC2 접속
Windows PC에서 Public EC2를 거쳐 Private EC2에 접속했습니다.

Windows PowerShell에서 다음 명령어를 사용했습니다.

ssh -o "ProxyCommand=ssh -i `"$HOME\Downloads\aws-devops-lab-key.pem`" -W %h:%p ec2-user@3.143.203.86" -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@10.10.2.105
정상적으로 접속하면 다음과 같이 Private EC2 Shell을 확인할 수 있습니다.

[ec2-user@ip-10-10-2-105 ~]$
이를 통해 다음 구조가 정상적으로 동작하는 것을 확인했습니다.

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
📸 SCREENSHOT 11 - Bastion SSH 접속

Windows PowerShell에서 Private EC2에 정상적으로 접속한 화면을 캡처합니다.

가능하면 다음 정보가 화면에 함께 보이도록 합니다.

Windows PowerShell

ProxyCommand

ec2-user@ip-10-10-2-105

이 화면은 Bastion Host 구성을 증명하는 중요한 포트폴리오 자료입니다.

13. Security Group 구성
이번 프로젝트에서는 Security Group을 이용하여 각 서버 간 접근을 제한했습니다.

13.1 Public EC2 Security Group
Name
aws-devops-vpc-sg
Public EC2는 Windows PC에서 SSH 접속할 수 있도록 SSH 22번 포트를 허용했습니다.

13.2 Private EC2 Security Group
Name
aws-devops-private-sg
Private EC2에는 다음과 같이 접근을 제한했습니다.

Type        Port    Source
SSH         22      aws-devops-vpc-sg
HTTP        80      aws-devops-alb-sg
즉,

SSH
Public EC2 Security Group
        │
        ▼
Private EC2

HTTP
ALB Security Group
        │
        ▼
Private EC2
인터넷 전체에 SSH를 개방하지 않고 Public EC2에서만 SSH 접속할 수 있도록 구성했습니다.

13.3 ALB Security Group
Name
aws-devops-alb-sg
외부 사용자가 ALB에 HTTP 요청을 보낼 수 있도록 다음과 같이 구성했습니다.

Type        Port    Source
HTTP        80      0.0.0.0/0
📸 SCREENSHOT 12 - Security Group

Security Group 화면에서 다음 규칙을 확인할 수 있도록 캡처합니다.

Private EC2 SG

SSH 22 → aws-devops-vpc-sg

HTTP 80 → aws-devops-alb-sg

ALB SG

HTTP 80 → 0.0.0.0/0

이 부분은 AWS 보안 구성의 핵심이므로 포트폴리오에 반드시 넣는 것을 권장합니다.

14. Application Load Balancer 구성
14.1 ALB 생성
Name
aws-devops-alb

Scheme
Internet-facing

IP address type
IPv4
ALB는 다음 Public Subnet 2개에 연결했습니다.

us-east-2a
aws-devops-public-subnet

us-east-2b
aws-devops-public-subnet-2
📸 SCREENSHOT 13 - ALB 구성

EC2 → Load Balancers에서 aws-devops-alb의 다음 내용이 보이도록 캡처합니다.

Internet-facing

IPv4

Availability Zone

Security Group

DNS Name

15. Target Group 구성
15.1 Target Group
Name
aws-devops-tg

Target Type
Instances

Protocol
HTTP

Port
80
Target으로 Private EC2를 등록했습니다.

Private EC2
10.10.2.105
Port 80
15.2 Health Check
Health Check 설정:

Protocol
HTTP

Port
Traffic Port

Path
/
처음에는 Private EC2의 HTTP 80 포트가 ALB Security Group으로부터 허용되지 않아 Health Check가 Unhealthy 상태였습니다.

Private EC2 Security Group에 다음 규칙을 추가한 후 정상적으로 변경되었습니다.

HTTP 80
Source:
aws-devops-alb-sg
최종 상태:

Healthy
📸 SCREENSHOT 14 - Target Group Healthy

EC2 → Target Groups → aws-devops-tg → Targets 화면에서 다음이 보이도록 캡처합니다.

Private EC2

Port 80

Health status: Healthy

이 화면은 반드시 캡처하는 것을 추천합니다.

ALB → Target Group → Private EC2 연결이 정상이라는 것을 한 장으로 증명할 수 있습니다.

16. ALB → Private EC2 테스트
ALB의 DNS 주소를 브라우저에서 접속하여 테스트했습니다.

Internet
   │
   │ HTTP :80
   ▼
ALB
   │
   │ HTTP :80
   ▼
Private EC2
   │
   ▼
Python HTTP Server
브라우저에서 다음 응답을 확인했습니다.

AWS DevOps Private EC2 - ALB TEST
이를 통해 다음 구성이 정상적으로 동작하는 것을 확인했습니다.

Internet
   ↓
Internet-facing ALB
   ↓
Target Group
   ↓
Private EC2
   ↓
Web Server
📸 SCREENSHOT 15 - ALB 접속 결과

브라우저에서 ALB DNS 주소로 접속한 화면을 캡처합니다.

주소창에 ALB DNS가 보이고 화면에 다음 문구가 표시되도록 캡처합니다.

AWS DevOps Private EC2 - ALB TEST

포트폴리오에서 가장 중요한 결과 화면 중 하나입니다.

17. Private EC2 인터넷 접근 테스트
Private EC2에서 외부 인터넷 접근 여부를 확인했습니다.

curl -I https://www.google.com
Private EC2에는 NAT Gateway가 구성되어 있지 않기 때문에 외부 인터넷으로 직접 통신할 수 없는 것을 확인했습니다.

이를 통해 Public Subnet과 Private Subnet의 네트워크 차이를 확인했습니다.

📸 SCREENSHOT 16 - Private EC2 인터넷 접근 제한

Private EC2에서 외부 인터넷 접근이 되지 않는 것을 확인한 터미널 화면입니다.

단, 포트폴리오에서는 오류 메시지를 너무 길게 보여주기보다는 명령어와 Timeout / 연결 실패 결과가 보이는 정도로 캡처하면 충분합니다.

18. 최종 네트워크 구조
전체적인 네트워크 구조는 다음과 같습니다.

                         ┌────────────────────┐
                         │     Windows PC     │
                         │     Local PC       │
                         └─────────┬──────────┘
                                   │
                               SSH :22
                                   │
                               Internet
                                   │
                                   ▼
                         ┌────────────────────┐
                         │     Public EC2     │
                         │   Bastion Host     │
                         └─────────┬──────────┘
                                   │
                               SSH :22
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                         AWS VPC                                 │
│                        10.10.0.0/16                              │
│                                                                  │
│  ┌─────────────────────────┐     ┌────────────────────────────┐ │
│  │     Public Subnet       │     │      Private Subnet        │ │
│  │     10.10.1.0/24        │     │      10.10.2.0/24         │ │
│  │                         │     │                            │ │
│  │  ┌───────────────────┐  │     │  ┌──────────────────────┐ │ │
│  │  │    Public EC2     │──┼────▶│  │     Private EC2      │ │ │
│  │  │   Bastion Host    │  │ SSH │  │     10.10.2.105      │ │ │
│  │  └───────────────────┘  │     │  │      HTTP :80        │ │ │
│  │                         │     │  └──────────────────────┘ │ │
│  └─────────────────────────┘     └────────────────────────────┘ │
│                                                                  │
│          ┌────────────────────────────────────┐                  │
│          │       Internet-facing ALB           │                  │
│          │       HTTP :80                      │                  │
│          └────────────────┬───────────────────┘                  │
│                           │                                      │
│                           │ HTTP :80                             │
│                           ▼                                      │
│                     Private EC2                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
📸 SCREENSHOT 17 - AWS VPC Resource 최종 화면

VPC Resource Map 또는 VPC Console에서 전체 네트워크 구성을 확인할 수 있다면 캡처합니다.

Architecture 이미지와 별도로 AWS Console의 실제 구성 화면을 보여주는 용도로 사용합니다.

19. 주요 명령어 정리
19.1 Public EC2 SSH 접속
ssh -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@3.143.203.86
19.2 Bastion을 통한 Private EC2 접속
ssh -o "ProxyCommand=ssh -i `"$HOME\Downloads\aws-devops-lab-key.pem`" -W %h:%p ec2-user@3.143.203.86" -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@10.10.2.105
19.3 Web Server 파일 생성
echo "AWS DevOps Private EC2 - ALB TEST" | sudo tee /tmp/index.html
19.4 Python HTTP Server 실행
sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
19.5 Web Server 테스트
curl http://localhost
19.6 Network 확인
ip addr
ip route
19.7 외부 인터넷 접근 테스트
curl -I https://www.google.com
20. Troubleshooting
20.1 Private EC2에서 dnf install이 되지 않는 문제
Private EC2에서 다음 명령어를 실행했을 때 정상적으로 패키지를 다운로드하지 못했습니다.

sudo dnf install nginx -y
원인은 Private Subnet에서 인터넷으로 나가는 NAT Gateway가 구성되어 있지 않기 때문입니다.

이번 프로젝트에서는 무료 실습을 위해 NAT Gateway를 사용하지 않았습니다.

대신 Python HTTP Server를 사용하여 Web Server를 구성했습니다.

sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
20.2 ALB Health Check가 Unhealthy인 문제
처음 ALB Target Group의 Health Check가 다음 상태였습니다.

Unhealthy
원인은 Private EC2 Security Group에서 ALB의 HTTP 요청을 허용하지 않았기 때문입니다.

Private EC2 Security Group에 다음 규칙을 추가했습니다.

HTTP
Port: 80
Source: aws-devops-alb-sg
이후 상태가 다음과 같이 변경되었습니다.

Healthy
20.3 Private EC2에 직접 SSH 접속이 되지 않는 문제
Private EC2에는 Public IP가 없기 때문에 Windows PC에서 직접 SSH 접속할 수 없습니다.

잘못된 접근:

Windows PC
    │
    │ SSH
    ▼
Private EC2
올바른 접근:

Windows PC
    │
    ▼
Public EC2
(Bastion Host)
    │
    ▼
Private EC2
20.4 Security Group 수정 오류
기존에 0.0.0.0/0으로 등록된 IPv4 규칙을 Security Group Source로 변경하려고 했을 때 다음과 같은 오류가 발생했습니다.

existing IPv4 CIDR rule cannot specify referenced group ID
기존 CIDR 규칙을 삭제한 후 Security Group을 Source로 사용하는 새로운 규칙을 생성하여 해결했습니다.

21. 보안 구성
이번 프로젝트에서는 다음과 같이 접근 범위를 분리했습니다.

Internet
   │
   │ HTTP :80
   ▼
ALB
   │
   │ HTTP :80
   ▼
Private EC2
ALB에서 Private EC2로 들어오는 HTTP 트래픽만 허용했습니다.

관리자 접근은 다음과 같이 구성했습니다.

Windows PC
   │
   │ SSH :22
   ▼
Public EC2
   │
   │ SSH :22
   ▼
Private EC2
Private EC2의 SSH 포트는 인터넷 전체에 공개하지 않고 Public EC2의 Security Group을 Source로 지정했습니다.

📸 SCREENSHOT 18 - 최종 Security Group 구성

Private EC2 Security Group의 최종 Inbound Rules 화면을 캡처합니다.

이 화면은 프로젝트의 보안 설계를 증명하는 자료로 사용합니다.

22. 비용 관리
이번 프로젝트는 AWS 실습 비용을 최소화하기 위해 NAT Gateway를 사용하지 않았습니다.

구성:

Internet
   │
   ▼
Internet Gateway
   │
   ▼
Public Subnet
Private Subnet:

Private Subnet
   │
   ▼
Private EC2
   │
   X
Internet
따라서 Private EC2가 인터넷으로 직접 나가는 구조는 구성하지 않았습니다.

실습 종료 후에는 사용하지 않는 AWS 리소스를 확인하여 불필요한 비용이 발생하지 않도록 관리해야 합니다.

23. 프로젝트 결과
이번 프로젝트에서는 AWS에서 다음과 같은 인프라를 직접 구축했습니다.

AWS VPC
  │
  ├── Public Subnet
  │      └── Public EC2
  │             └── Bastion Host
  │
  ├── Private Subnet
  │      └── Private EC2
  │             └── Web Server
  │
  ├── Internet Gateway
  │
  ├── Route Table
  │
  ├── Security Group
  │
  └── Application Load Balancer
           │
           └── Target Group
                  │
                  └── Private EC2
최종적으로 다음 두 가지 경로가 정상적으로 동작하는 것을 확인했습니다.

관리자 접근
Windows PC
    ↓
Internet
    ↓
Public EC2
    ↓
Private EC2
사용자 서비스 접근
Internet
    ↓
Internet-facing ALB
    ↓
Target Group
    ↓
Private EC2
    ↓
Web Server
24. 프로젝트를 통해 학습한 내용
AWS Network
VPC

CIDR

Public Subnet

Private Subnet

Availability Zone

Internet Gateway

Route Table

AWS Compute
EC2

Public EC2

Private EC2

Bastion Host

AWS Security
Security Group

Port 기반 접근 제어

Security Group Reference

Public / Private 네트워크 분리

AWS Load Balancing
Application Load Balancer

Internet-facing ALB

Target Group

Health Check

ALB → Private EC2 연결

Linux
SSH

ProxyCommand

ip route

ip addr

curl

Python HTTP Server

nohup

25. 프로젝트 핵심 정리
이번 프로젝트의 핵심은 단순히 EC2를 생성하는 것이 아니라 AWS의 기본적인 네트워크 구조를 직접 구축하고 각 구성 요소의 역할을 확인한 것입니다.

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
특히 Public EC2와 Private EC2를 분리하고 Bastion Host를 통해 Private EC2에 접근하는 구조를 직접 구현했습니다.

또한 Internet-facing ALB를 통해 인터넷에서 Private EC2의 Web Server까지 HTTP 요청이 전달되는 구조를 구축하고 Health Check를 통해 Target 상태를 확인했습니다.

26. 향후 확장 계획
이번 프로젝트를 기반으로 다음 단계의 DevOps 프로젝트로 확장할 예정입니다.

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
기존 Linux / Server / Security 운영 경험을 기반으로 AWS Cloud, Infrastructure as Code, Container, CI/CD, Kubernetes 및 DevSecOps 영역까지 역량을 확장하는 것을 목표로 합니다.

27. Screenshot Checklist
포트폴리오 작성 시 다음 화면을 캡처하여 screenshots/ 폴더에 저장합니다.

screenshots/
├── 01-architecture.png
├── 02-region.png
├── 03-vpc.png
├── 04-subnets.png
├── 05-internet-gateway.png
├── 06-public-route-table.png
├── 07-private-route-table.png
├── 08-public-ec2.png
├── 09-private-ec2.png
├── 10-private-web-server.png
├── 11-bastion-ssh.png
├── 12-security-groups.png
├── 13-alb.png
├── 14-target-group-healthy.png
├── 15-alb-test.png
├── 16-private-internet-test.png
├── 17-vpc-resource-map.png
└── 18-final-security-group.png
우선순위가 높은 Screenshot
포트폴리오에서 특히 중요하게 사용할 화면은 다음과 같습니다.

[필수]

01. 전체 Architecture
02. VPC
03. Subnets
08. Public EC2
09. Private EC2
11. Bastion SSH
12. Security Groups
13. ALB
14. Target Group - Healthy
15. ALB 접속 결과

[있으면 좋은 자료]

05. Internet Gateway
06. Public Route Table
07. Private Route Table
16. Private EC2 인터넷 접근 제한
17. VPC Resource Map
28. 프로젝트 한 줄 요약
AWS VPC 환경에서 Public / Private Subnet을 분리하고 Bastion Host를 통한 Private EC2 관리 및 Internet-facing ALB를 통한 Private Web Server 서비스를 구현했습니다.
