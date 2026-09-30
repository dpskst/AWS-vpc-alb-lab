# Linux 명령어 기록

## 시스템 확인

```bash
hostname
ip -4 addr
ip route
```

## 네트워크 테스트

```bash
curl http://localhost
curl -I https://www.google.com
```

## 포트 확인

```bash
sudo ss -lntp
sudo ss -lntp | grep :80
```

## 웹 서버 실행

```bash
echo "AWS DevOps Private EC2 - ALB TEST" | sudo tee /tmp/index.html

sudo nohup python3 -m http.server 80 --directory /tmp > /tmp/http.log 2>&1 &
```

## 웹 서버 확인

```bash
curl http://localhost
```

## 프로세스 종료

```bash
sudo pkill -f "python3 -m http.server"
```

## AWS Linux 패키지 설치 시도

```bash
sudo dnf install nginx -y
```

Private Subnet에 NAT Gateway가 없으므로 인터넷 저장소 접근이 되지 않는 상황을 확인했다.
