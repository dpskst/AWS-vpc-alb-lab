# Troubleshooting 기록

## 1. Xshell SSH 인증 문제

기존 Xshell 환경에서 EC2 Key Pair 인증 문제가 발생했다.

대응:

- Windows PowerShell OpenSSH 사용
- AWS에서 발급한 PEM 키로 SSH 접속

## 2. Private EC2에서 Nginx 설치 지연

실행:

```bash
sudo dnf install nginx -y
```

Private Subnet에서 인터넷 접근이 불가능하여 패키지 저장소 접근이 지연됐다.

이번 실습에서는 NAT Gateway를 추가하지 않고 Python 내장 HTTP 서버로 대체했다.

## 3. ALB Target Health Check 실패

증상:

```text
Target: Unhealthy
Health checks failed
```

원인:

Private EC2 Security Group에서 ALB의 HTTP 요청을 허용하지 않았다.

해결:

```text
aws-devops-private-sg

HTTP
TCP 80
Source: aws-devops-alb-sg
```

추가 후:

```text
Unhealthy → Healthy
```

로 변경됐다.

## 4. ProxyJump 명령 차이

다음 ProxyCommand 방식으로 Public EC2를 경유한 Private EC2 접속을 성공시켰다.

```powershell
ssh -o "ProxyCommand=ssh -i `"$HOME\Downloads\aws-devops-lab-key.pem`" -W %h:%p ec2-user@3.143.203.86" -i "$HOME\Downloads\aws-devops-lab-key.pem" ec2-user@10.10.2.105
```

Private EC2의 Public IP가 없기 때문에 Bastion을 경유하는 구조를 확인했다.
