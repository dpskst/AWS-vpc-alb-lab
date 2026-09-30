# Network 구성

## VPC

```text
Name: aws-devops-vpc
CIDR: 10.10.0.0/16
Region: us-east-2
```

## Subnet

| Name | CIDR | AZ | 용도 |
|---|---|---|---|
| aws-devops-public-subnet | 10.10.1.0/24 | us-east-2a | Public EC2 |
| aws-devops-private-subnet | 10.10.2.0/24 | us-east-2a | Private EC2 |
| aws-devops-public-subnet-2 | 10.10.3.0/24 | us-east-2b | ALB |

## Internet Gateway

```text
aws-devops-igw
```

VPC:

```text
aws-devops-vpc
```

## Public Route Table

```text
aws-devops-public-rt
```

Route:

```text
0.0.0.0/0 → Internet Gateway
```

Associated subnets:

```text
aws-devops-public-subnet
aws-devops-public-subnet-2
```

## Private Route Table

```text
aws-devops-private-rt
```

Private subnet:

```text
aws-devops-private-subnet
```

이번 실습에서는 NAT Gateway를 추가하지 않았다.
