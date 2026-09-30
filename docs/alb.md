# ALB 구성

## Load Balancer

```text
Name: aws-devops-alb
Type: Application Load Balancer
Scheme: Internet-facing
IP type: IPv4
```

ALB는 다음 두 Public Subnet에 배치했다.

```text
us-east-2a
aws-devops-public-subnet

us-east-2b
aws-devops-public-subnet-2
```

## Target Group

```text
Name: aws-devops-tg
Protocol: HTTP
Port: 80
Target Type: Instances
```

Target:

```text
aws-devops-private-ec2
10.10.2.105
```

## Health Check

```text
Protocol: HTTP
Path: /
Port: traffic-port
```

처음 `Unhealthy` 상태가 발생했으나 Private EC2 Security Group에 ALB Security Group으로부터 HTTP/80을 허용한 후 `Healthy`로 변경됐다.

## 최종 요청 흐름

```text
Client
  ↓
ALB :80
  ↓
Target Group
  ↓
Private EC2 :80
  ↓
Python HTTP Server
```
