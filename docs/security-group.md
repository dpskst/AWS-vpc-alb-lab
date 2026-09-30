# Security Group 구성

## Public EC2

Public EC2는 Bastion Host 역할을 수행한다.

주요 규칙:

```text
SSH :22
Source: 관리용 접근 범위
```

## Private EC2

Private EC2는 인터넷에 직접 노출하지 않는다.

### SSH

```text
Type: SSH
Port: 22
Source: aws-devops-vpc-sg
```

### HTTP

```text
Type: HTTP
Port: 80
Source: aws-devops-alb-sg
```

즉, Private EC2의 HTTP 서비스는 ALB에서만 접근하도록 제한한다.

## ALB

```text
aws-devops-alb-sg
```

Inbound:

```text
HTTP :80
Source: 0.0.0.0/0
```

ALB가 인터넷에서 HTTP 요청을 받아야 하기 때문에 외부 HTTP 접근을 허용한다.

## 핵심 원칙

```text
Internet
   ↓
ALB :80
   ↓
Private EC2 :80

Internet
   X
   ↓
Private EC2 :22
```
