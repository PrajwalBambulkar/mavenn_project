# Infrastructure Documentation
<img width="1643" height="885" alt="image" src="https://github.com/user-attachments/assets/aa872f59-c096-4a98-818b-909356ce8c6e" />


## VPC (Virtual Private Cloud)

A VPC (Virtual Private Cloud) is a logically isolated private network in AWS where you can launch and manage resources securely. It allows you to define your own IP range, subnets, routing, and security controls.

### VPC Configuration

| S/N | VPC Name | VPC ID | IPv4 CIDR |
|-----|----------|--------|-----------|
| 1 | `bidd-uat-vpc` | `vpc-0b1f4a3ecc95086e2` | `10.70.0.0/16` |
| 2 | `bidd-staging-vpc` | `vpc-01aa775625bdbc866` | `10.80.0.0/16` |
| 3 | `bidd-prod-vpc` | `vpc-0ba146c7ffff6ddf4` | `10.90.0.0/16` |

---

## Routing Tables

### UAT Environment

| S/N | Route Table Name | Type | Associated Subnet | AZ | Target | VPC Name | ENV |
|-----|------------------|------|-------------------|-------|--------|----------|-----|
| 1 | `bidd-uat-rtb-public` | Public | `bidd-uat-subnet-public1-ap-south-1a` | `ap-south-1a` | IGW (`bidd-uat-igw`) | `bidd-uat-vpc` | UAT |
| 2 | `bidd-uat-rtb-public` | Public | `bidd-uat-subnet-public2-ap-south-1b` | `ap-south-1b` | IGW (`bidd-uat-igw`) | `bidd-uat-vpc` | UAT |
| 3 | `bidd-uat-rtb-private1-ap-south-1a` | Private | `bidd-uat-subnet-private1-ap-south-1a` | `ap-south-1a` | NAT (`bidd-uat-regional-nat`) | `bidd-uat-vpc` | UAT |
| 4 | `bidd-uat-rtb-private2-ap-south-1b` | Private | `bidd-uat-subnet-private2-ap-south-1b` | `ap-south-1b` | NAT (`bidd-uat-regional-nat`) | `bidd-uat-vpc` | UAT |

#### Routes Configuration - bidd-uat-rtb-public

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `igw-08e2396be919a0246` |
| 2 | `10.70.0.0/16` | `local` |

#### Routes Configuration - bidd-uat-rtb-private2-ap-south-1b

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `nat-1d76874913dfc8bb6` |
| 2 | `10.0.0.0/16` | `pcx-0761b6447a5af5541` |
| 3 | `10.70.0.0/16` | `local` |
| 4 | `192.168.248.0/21` | `pcx-0f9c6969527912b03` |

#### Routes Configuration - bidd-uat-rtb-private1-ap-south-1a

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `nat-1d76874913dfc8bb6` |
| 2 | `10.0.0.0/16` | `pcx-0761b6447a5af5541` |
| 3 | `10.70.0.0/16` | `local` |
| 4 | `10.100.0.0/16` | `pcx-0bfb406f33d3f9d39` |
| 5 | `192.168.248.0/21` | `pcx-0f9c6969527912b03` |

---

### Staging Environment

| S/N | Route Table Name | Type | Associated Subnet | AZ | Target | VPC Name | ENV |
|-----|------------------|------|-------------------|-------|--------|----------|---------|
| 1 | `bidd-staging-rtb-public` | Public | `bidd-staging-subnet-public1-ap-south-1a` | `ap-south-1a` | IGW (`bidd-staging-igw`) | `bidd-staging-vpc` | STAGING |
| 2 | `bidd-staging-rtb-public` | Public | `bidd-staging-subnet-public2-ap-south-1b` | `ap-south-1b` | IGW (`bidd-staging-igw`) | `bidd-staging-vpc` | STAGING |
| 3 | `bidd-staging-rtb-private1-ap-south-1a` | Private | `bidd-staging-subnet-private1-ap-south-1a` | `ap-south-1a` | NAT (`bidd-staging-regional-nat`) | `bidd-staging-vpc` | STAGING |
| 4 | `bidd-staging-rtb-private2-ap-south-1b` | Private | `bidd-staging-subnet-private2-ap-south-1b` | `ap-south-1b` | NAT (`bidd-staging-regional-nat`) | `bidd-staging-vpc` | STAGING |

#### Routes Configuration - bidd-staging-rtb-public

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | Internet Gateway (`bidd-staging-igw`) |
| 2 | `10.80.0.0/16` | `local` |

#### Routes Configuration - bidd-staging-rtb-private1-ap-south-1a

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | NAT Gateway (`bidd-staging-regional-nat`) |
| 2 | `10.80.0.0/16` | `local` |
| 3 | `10.0.0.0/16` | `pcx-024ff73829ebc9b74` |
| 4 | `10.100.0.0/16` | `pcx-0a2f5318908894785` |
| 5 | `192.168.248.0/21` | `pcx-02ab332191c3e1c47` |

#### Routes Configuration - bidd-staging-rtb-private2-ap-south-1b

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `nat-105148f1e51dc6e7e` |
| 2 | `10.0.0.0/16` | `pcx-024ff73829ebc9b74` |
| 3 | `10.80.0.0/16` | `local` |
| 4 | `192.168.248.0/21` | `pcx-02ab332191c3e1c47` |

---

### Production Environment

| S/N | Route Table Name | Type | Associated Subnet | AZ | Target | VPC Name |
|-----|------------------|------|-------------------|-------|--------|----------|
| 1 | `bidd-prod-rtb-public` | Public | `bidd-prod-subnet-public1-ap-south-1a` | `ap-south-1a` | IGW (`bidd-prod-igw`) | `bidd-prod-vpc` |
| 2 | `bidd-prod-rtb-public` | Public | `bidd-prod-subnet-public2-ap-south-1b` | `ap-south-1b` | IGW (`bidd-prod-igw`) | `bidd-prod-vpc` |
| 3 | `bidd-prod-rtb-private1-ap-south-1a` | Private | `bidd-prod-subnet-private1-ap-south-1a` | `ap-south-1a` | NAT (`bidd-prod-regional-nat`) | `bidd-prod-vpc` |
| 4 | `bidd-prod-rtb-private2-ap-south-1b` | Private | `bidd-prod-subnet-private2-ap-south-1b` | `ap-south-1b` | NAT (`bidd-prod-regional-nat`) | `bidd-prod-vpc` |

#### Routes Configuration - bidd-prod-rtb-public

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | Internet Gateway (`bidd-prod-igw`) |
| 2 | `10.90.0.0/16` | `local` |

#### Routes Configuration - bidd-prod-rtb-private1-ap-south-1a

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `nat-1cdee5a37f1ea8ca2` |
| 2 | `10.0.0.0/16` | `pcx-05402c462d8fec793` |
| 3 | `10.90.0.0/16` | `local` |
| 4 | `10.100.0.0/16` | `pcx-0b8b49e9767982f89` |
| 5 | `192.168.248.0/21` | `pcx-01dea8349861b7e63` |

#### Routes Configuration - bidd-prod-rtb-private2-ap-south-1b

| S/N | Destination CIDR | Target |
|-----|------------------|--------|
| 1 | `0.0.0.0/0` | `nat-1cdee5a37f1ea8ca2` |
| 2 | `10.0.0.0/16` | `pcx-05402c462d8fec793` |
| 3 | `10.90.0.0/16` | `local` |
| 4 | `10.100.0.0/16` | `pcx-0b8b49e9767982f89` |
| 5 | `192.168.248.0/21` | `pcx-01dea8349861b7e63` |

---

## NAT (Network Address Translation)

NAT allows instances in a private subnet to access the internet while preventing inbound internet traffic to them.

| S/N | Name | NAT ID | Route Table ID | IPv4 | VPC |
|-----|------|--------|----------------|------|-----|
| 1 | `bidd-uat-regional-nat` | `nat-1d76874913dfc8bb6` | `rtb-0623edd2b67f1d398` | `3.108.176.63` | `vpc-0b1f4a3ecc95086e2` |
| 2 | `bidd-staging-regional-nat` | `nat-105148f1e51dc6e7e` | `rtb-005ba9c89a71a0c06` | `13.200.188.24` | `vpc-01aa775625bdbc866` |
| 3 | `bidd-prod-regional-nat` | `nat-1cdee5a37f1ea8ca2` | `rtb-032b4d028c71515ef` | `13.204.162.227` | `vpc-0ba146c7ffff6ddf4` |

---

## IGW (Internet Gateway)

An Internet Gateway (IGW) is an AWS-managed component that is attached to a VPC to enable internet connectivity. It allows public subnet resources, such as EC2 instances with public IPs, to send and receive traffic from the internet and works with route tables to route traffic outside the VPC securely.

| S/N | Name | IGW ID | VPC ID |
|-----|------|--------|--------|
| 1 | `bidd-uat-igw` | `igw-08e2396be919a0246` | `vpc-0b1f4a3ecc95086e2` |
| 2 | `bidd-staging-igw` | `igw-09d4e19ae3bed1de0` | `vpc-01aa775625bdbc866` |
| 3 | `bidd-prod-igw` | `igw-0e9990549df1a2783` | `vpc-0ba146c7ffff6ddf4` |

---

## VPC Endpoints

### UAT Environment

| S/N | Name | VPC Endpoint ID | VPC ID |
|-----|------|-----------------|--------|
| 1 | `uat-ssm-endpoint` | `vpce-0751bb4b2ec44c22e` | `vpc-0b1f4a3ecc95086e2` |
| 2 | `uat-ssmmessages-endpoint` | `vpce-09ead38be8e3c183a` | `vpc-0b1f4a3ecc95086e2` |
| 3 | `uat-ec2msg-endpoint` | `vpce-007c1d33200aa4f4e` | `vpc-0b1f4a3ecc95086e2` |

### Staging Environment

| S/N | Name | VPC Endpoint ID | VPC ID |
|-----|------|-----------------|--------|
| 1 | `staging-ssm-endpoint` | `vpce-0b6c384e50ca88975` | `vpc-01aa775625bdbc866` |
| 2 | `staging-ssmmessages-endpoint` | `vpce-0d14c04c15f1a148d` | `vpc-01aa775625bdbc866` |
| 3 | `staging-ec2msg-endpoint` | `vpce-09ddeb1a93aac6895` | `vpc-01aa775625bdbc866` |

### Production Environment

| S/N | Name | VPC Endpoint ID | VPC ID |
|-----|------|-----------------|--------|
| 1 | `prod-ssm-endpoint` | `vpce-08623dd6f1f37e065` | `vpc-0ba146c7ffff6ddf4` |
| 2 | `prod-ssmmessages-endpoint` | `vpce-042d39681230fd098` | `vpc-0ba146c7ffff6ddf4` |
| 3 | `prod-ec2msg-endpoint` | `vpce-0caa85ad654348679` | `vpc-0ba146c7ffff6ddf4` |

---

## Security Groups

A Security Group is a virtual firewall in AWS that controls inbound and outbound traffic for resources like EC2 instances. It works at the instance level and allows traffic only if it is explicitly permitted by defined rules.

### UAT Environment

| S/N | Name | Security Group ID | VPC ID |
|-----|------|-------------------|--------|
| 1 | `uat-biddEasy-app-be-sg` | `sg-069e32c90baab413e` | `vpc-0b1f4a3ecc95086e2` |
| 2 | `uat-biddEasy-alb-sg` | `sg-00c0211b331681f00` | `vpc-0b1f4a3ecc95086e2` |
| 3 | `uat-biddEasy-app-fe-sg` | `sg-05af5fbadce17ccc6` | `vpc-0b1f4a3ecc95086e2` |
| 4 | `uat-biddEasy-vpc-endpoint-sg` | `sg-082d9aa6b1b65ad74` | `vpc-0b1f4a3ecc95086e2` |

---

### uat-biddEasy-app-be-sg

#### Inbound Traffic

| Rule Type | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| SSH | TCP | 22 | `10.90.0.0/16` | Admin access from internal network |
| SSH | TCP | 22 | `10.100.0.0/16` | Admin access from internal network |
| Custom TCP | TCP | 3515 | `sg-00c0211b331681f00` | Application service communication |
| Custom TCP | TCP | 3000 | `sg-00c0211b331681f00` | Web application access |
| Custom TCP | TCP | 3002 | `sg-00c0211b331681f00` | Internal service / API access |

#### Outbound Traffic

| Rule Type | Protocol | Port | Destination | Purpose |
|-----------|----------|------|-------------|---------|
| Custom TCP | TCP | 587 | `0.0.0.0/0` | Outbound email via AWS SES (SMTP) |
| Custom TCP | TCP | 27017 | `192.168.248.0/21` | Application access to MongoDB (Atlas / DB network) |
| HTTP | TCP | 80 | `0.0.0.0/0` | Outbound internet HTTP traffic |
| SSH | TCP | 22 | `20.207.73.82/32` | SSH access for GitHub repository clone |
| DNS (TCP) | TCP | 53 | `10.70.0.2/32` | DNS resolution (fallback) |
| DNS (UDP) | UDP | 53 | `10.70.0.2/32` | DNS resolution |
| HTTPS | TCP | 443 | `sg-082d9aa6b1b65ad74` | AWS SSM connectivity |
| HTTPS | TCP | 443 | `0.0.0.0/0` | Outbound internet HTTPS traffic |

---

### uat-biddEasy-alb-sg

#### Inbound Traffic

| Rule Type | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| HTTPS | TCP | 443 | `0.0.0.0/0` | For HTTPS requests |

#### Outbound Traffic

| Rule Type | Protocol | Port | Destination | Purpose |
|-----------|----------|------|-------------|---------|
| Custom TCP | TCP | 3002 | `sg-069e32c90baab413e` | For Exchange Service |
| Custom TCP | TCP | 3000 | `sg-069e32c90baab413e` | For BE services |
| Custom TCP | TCP | 3000 | `sg-05af5fbadce17ccc6` | For FE services |

---

### uat-biddEasy-app-fe-sg

#### Inbound Traffic

| Rule Type | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| Custom TCP | TCP | 3000 | `sg-00c0211b331681f00` | Web Requests from UAT ALB |
| SSH | TCP | 22 | `10.90.0.0/16` | SSH from bidd Prod VPC |
| SSH | TCP | 22 | `10.100.0.0/16` | SSH from old bidd Prod VPC |

#### Outbound Traffic

| Rule Type | Protocol | Port | Destination | Purpose |
|-----------|----------|------|-------------|---------|
| HTTPS | TCP | 443 | `sg-082d9aa6b1b65ad74` | For SSM connect |
| SSH | TCP | 22 | `20.207.73.82/32` | SSH clone from GitHub |
| HTTPS | TCP | 443 | `0.0.0.0/0` | Internet requests goes via NAT only for port 443 |
| DNS (UDP) | UDP | 53 | `10.70.0.2/32` | DNS |
| DNS (TCP) | TCP | 53 | `10.70.0.2/32` | DNS fallback |
| HTTP | TCP | 80 | `0.0.0.0/0` | Internet requests goes via NAT only for port 80 |

---

### uat-biddEasy-vpc-endpoint-sg

#### Inbound Traffic

| Rule Type | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| HTTPS | TCP | 443 | `sg-05af5fbadce17ccc6` | For SSM connect [Frontend EC2] |
| HTTPS | TCP | 443 | `sg-069e32c90baab413e` | For SSM connect [Backend EC2] |

#### Outbound Traffic

| Rule Type | Protocol | Port | Destination | Purpose |
|-----------|----------|------|-------------|---------|
| All | ALL | ALL | `0.0.0.0/0` | Required for AWS SSM endpoint communication |

---

## Staging Environment

| Name | Security Group ID | Security Group Name | VPC ID | Description | Owner |
|------|-------------------|---------------------|--------|-------------|-------|
| `staging-biddEasy-app-be-sg` | `sg-0c31cb184a901dea9` | `staging-biddEasy-app-be-sg` | `vpc-01aa775625bdbc866` | `staging-biddEasy-app-be-sg` | `851721469379` |
| `staging-biddEasy-vpc-endpoint-sg` | `sg-0b13e44deabb8aee4` | `staging-biddEasy-vpc-endpoint-sg` | `vpc-01aa775625bdbc866` | `staging-biddEasy-vpc-endpoint-sg` | `851721469379` |
| `staging-portald-app-sg` | `sg-0c12c1b87cb0ef204` | `staging-portald-app-sg` | `default` | `staging-portald-app-sg` | `851721469379` |
| `staging-biddEasy-alb-sg` | `sg-04c9e125628a0f9043` | `staging-biddEasy-alb-sg` | `vpc-01aa775625bdbc866` | `staging-biddEasy-alb-sg` | `851721469379` |
| `staging-biddEasy-app-fe-sg` | `sg-01e4a7758325d8afc` | `staging-biddEasy-app-fe-sg` | `vpc-01aa775625bdbc866` | `staging-biddEasy-app-fe-sg` | `851721469379` |

---

### staging-biddEasy-app-fe-sg

#### Inbound Traffic

<img width="1190" alt="staging-fe-inbound" src="https://github.com/user-attachments/assets/78b087e9-d02b-4e67-bd90-89dc110c144b" />

#### Outbound Traffic

<img width="1189" alt="staging-fe-outbound" src="https://github.com/user-attachments/assets/ac1171e5-9d01-484e-9725-11c29395060c" />

---

### staging-biddEasy-vpc-endpoint-sg

#### Inbound Traffic

<img width="1188" alt="staging-vpc-endpoint-inbound" src="https://github.com/user-attachments/assets/dee79cb1-e0bf-43fb-b0f7-156272515c91" />

#### Outbound Traffic

<img width="1188" alt="staging-vpc-endpoint-outbound" src="https://github.com/user-attachments/assets/48e469cd-3cdb-4ec4-9661-dfeeed85c804" />

---

### staging-biddEasy-bastion-sg

#### Inbound Traffic

<img width="1188" alt="staging-bastion-inbound" src="https://github.com/user-attachments/assets/4ee2f3ae-9284-4b35-bdad-709f7a5b804c" />

#### Outbound Traffic

<img width="1188" alt="staging-bastion-outbound" src="https://github.com/user-attachments/assets/0b1537ac-0a48-4c77-8638-d89c54276acf" />

---

### staging-biddEasy-alb-sg

#### Inbound Traffic

<img width="1188" alt="staging-alb-inbound" src="https://github.com/user-attachments/assets/002e2680-dc8c-4c29-9e72-3bf8167e8b4e" />

#### Outbound Traffic

<img width="1188" alt="staging-alb-outbound" src="https://github.com/user-attachments/assets/0394c3f1-4176-4076-950a-13513f4bc4ba" />

---

## Production Environment

<img width="1188" alt="prod-security-groups-overview" src="https://github.com/user-attachments/assets/f519ae17-6c67-406d-a32d-2c3eb6d03f25" />

---

### prod-biddEasy-app-fe-sg

#### Inbound Traffic

<img width="1204" alt="prod-fe-inbound" src="https://github.com/user-attachments/assets/c7d55113-5ee1-483f-9eeb-97a257f41b44" />

#### Outbound Traffic

<img width="1199" alt="prod-fe-outbound" src="https://github.com/user-attachments/assets/ecdff417-3fa8-4e6c-918d-a1d562ad9989" />

---

### prod-biddEasy-bastion-sg

#### Inbound Traffic

<img width="1199" alt="prod-bastion-inbound" src="https://github.com/user-attachments/assets/d3c78ea0-58a9-4a3a-a551-f20b43cc7254" />

#### Outbound Traffic

<img width="1199" alt="prod-bastion-outbound" src="https://github.com/user-attachments/assets/5f1fba01-0c0c-4401-9a3a-28171c288590" />

---

### prod-biddEasy-redis-sg

#### Inbound Traffic

<img width="1199" alt="prod-redis-inbound" src="https://github.com/user-attachments/assets/156099c8-1ae8-4c07-9ff2-08654b76db09" />

#### Outbound Traffic

<img width="1199" alt="prod-redis-outbound" src="https://github.com/user-attachments/assets/165d3609-a426-40df-b6d6-f2dbc8988539" />

---

### prod-biddEasy-alb-sg

#### Inbound Traffic

<img width="1199" alt="prod-alb-inbound" src="https://github.com/user-attachments/assets/fa3e98c5-c4df-49ff-a807-25f52a3dfc74" />

#### Outbound Traffic

<img width="1199" alt="prod-alb-outbound" src="https://github.com/user-attachments/assets/4b990d57-18d7-403f-bffa-4168f0c18e0c" />

---

### prod-biddEasy-app-be-sg

#### Inbound Traffic

<img width="1199" alt="prod-be-inbound" src="https://github.com/user-attachments/assets/4eda1b73-f95f-4ad5-8c09-c2c7934c0294" />

#### Outbound Traffic

<img width="1199" alt="prod-be-outbound" src="https://github.com/user-attachments/assets/0127cae8-c6f8-4b2f-b409-0869026fa347" />

---

### prod-biddEasy-vpc-endpoint-sg

#### Inbound Traffic

<img width="1199" alt="prod-vpc-endpoint-inbound" src="https://github.com/user-attachments/assets/a488b8e4-34fd-41f2-bebe-ee370dbc9fd1" />

#### Outbound Traffic

<img width="1199" alt="prod-vpc-endpoint-outbound" src="https://github.com/user-attachments/assets/0a23f8fa-bdd0-4849-b4f8-9173c8853312" />

---

## NACL (Network Access Control List)

A NACL (Network Access Control List) is a stateless network firewall in AWS that controls inbound and outbound traffic at the subnet level. It uses allow and deny rules to provide an extra layer of security for subnets.

### UAT Environment

| Name | VPC ID |
|------|--------|
| `uat-biddEasy-private-nacl` | `vpc-0b1f4a3ecc95086e2` |
| `uat-biddEasy-public-nacl` | `vpc-0b1f4a3ecc95086e2` |

---

### uat-biddEasy-private-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 3000 | `10.70.1.0/24` | Allow |
| 130 | Custom TCP | TCP | 3000 | `10.70.2.0/24` | Allow |
| 140 | Custom TCP | TCP | 3002 | `10.70.1.0/24` | Allow |
| 150 | Custom TCP | TCP | 3002 | `10.70.2.0/24` | Allow |
| 160 | Custom TCP | TCP | 3515 | `10.70.1.0/24` | Allow |
| 170 | Custom TCP | TCP | 3515 | `10.70.2.0/24` | Allow |
| 180 | SSH | TCP | 22 | `10.70.0.0/16` | Allow |
| 190 | SSH | TCP | 22 | `10.90.0.0/16` | Allow |
| 200 | SSH | TCP | 22 | `10.100.0.0/16` | Allow |
| 210 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| * | All | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules

| Rule No | Type | Protocol | Port | Destination | Allow/Deny |
|---------|------|----------|------|-------------|------------|
| 100 | HTTPS | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 587 | `0.0.0.0/0` | Allow |
| 130 | Custom TCP | TCP | 27017 | `192.168.248.0/21` | Allow |
| 140 | HTTPS | TCP | 443 | `10.70.0.0/16` | Allow |
| 150 | DNS (TCP) | TCP | 53 | `10.70.0.2/32` | Allow |
| 160 | DNS (UDP) | UDP | 53 | `10.70.0.2/32` | Allow |
| 170 | SSH | TCP | 22 | `20.207.73.82/32` | Allow |
| 180 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 190 | SSH | TCP | 22 | `10.70.11.0/24` | Allow |
| 200 | SSH | TCP | 22 | `10.70.12.0/24` | Allow |
| * | All | All | All | `0.0.0.0/0` | Deny |

---

### uat-biddEasy-public-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 120 | SSH | TCP | 22 | `0.0.0.0/0` | Allow |
| * | All | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules
<img width="1199" height="367" alt="image" src="https://github.com/user-attachments/assets/463a8772-2ef1-44a0-91b1-16118d161f96" />


---
### Staging Environment

| Name | VPC ID |
|------|--------|
| `staging-biddEasy-default-nacl` | `vpc-01aa775625bdbc866` |
| `staging-biddEasy-private-nacl` | `vpc-01aa775625bdbc866` |
| `staging-biddEasy-public-nacl` | `vpc-01aa775625bdbc866` |

---

### staging-biddEasy-private-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP (80) | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 3000 | `10.80.1.0/24` | Allow |
| 130 | Custom TCP | TCP | 3000 | `10.80.2.0/24` | Allow |
| 140 | Custom TCP | TCP | 3515 | `10.80.1.0/24` | Allow |
| 150 | Custom TCP | TCP | 3515 | `10.80.1.0/24` | Allow |
| 160 | SSH (22) | TCP | 22 | `10.80.0.0/16` | Allow |
| 170 | SSH (22) | TCP | 22 | `10.90.0.0/16` | Allow |
| 180 | SSH (22) | TCP | 22 | `10.100.0.0/16` | Allow |
| 190 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules

| Rule No | Type | Protocol | Port | Destination | Allow/Deny |
|---------|------|----------|------|-------------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP (80) | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 587 | `0.0.0.0/0` | Allow |
| 130 | Custom TCP | TCP | 27017 | `192.168.248.0/21` | Allow |
| 140 | HTTPS (443) | TCP | 443 | `10.80.0.0/16` | Allow |
| 150 | DNS (TCP) (53) | TCP | 53 | `10.80.0.2/32` | Allow |
| 160 | DNS (UDP) (53) | UDP | 53 | `10.80.0.2/32` | Allow |
| 170 | SSH (22) | TCP | 22 | `20.207.73.82/32` | Allow |
| 180 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 190 | SSH (22) | TCP | 22 | `10.80.11.0/24` | Allow |
| 200 | SSH (22) | TCP | 22 | `10.80.12.0/24` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

---

### staging-biddEasy-public-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 120 | SSH (22) | TCP | 22 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules

| Rule No | Type | Protocol | Port | Destination | Allow/Deny |
|---------|------|----------|------|-------------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | Custom TCP | TCP | 3000 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 3000 | `0.0.0.0/0` | Allow |
| 130 | Custom TCP | TCP | 3515 | `0.0.0.0/0` | Allow |
| 140 | Custom TCP | TCP | 3515 | `0.0.0.0/0` | Allow |
| 150 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 160 | SSH (22) | TCP | 22 | `0.0.0.0/0` | Allow |
| 170 | SSH (22) | TCP | 22 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

---
### Production Environment

| Name | VPC ID |
|------|--------|
| `prod-biddEasy-default-nacl` | `vpc-0ba146c7ffff6ddf4` |
| `prod-biddEasy-private-nacl` | `vpc-0ba146c7ffff6ddf4` |
| `prod-biddEasy-public-nacl` | `vpc-0ba146c7ffff6ddf4` |

---

### prod-biddEasy-private-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP (80) | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 3000 | `0.0.0.0/0` | Allow |
| 130 | Custom TCP | TCP | 3000 | `0.0.0.0/0` | Allow |
| 140 | Custom TCP | TCP | 3002 | `0.0.0.0/0` | Allow |
| 150 | Custom TCP | TCP | 3002 | `0.0.0.0/0` | Allow |
| 160 | Custom TCP | TCP | 3515 | `0.0.0.0/0` | Allow |
| 170 | Custom TCP | TCP | 3515 | `0.0.0.0/0` | Allow |
| 180 | SSH (22) | TCP | 22 | `10.90.0.0/16` | Allow |
| 190 | SSH (22) | TCP | 22 | `10.100.0.0/16` | Allow |
| 200 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 210 | Custom TCP | TCP | 6379 | `10.90.11.0/24` | Allow |
| 220 | Custom TCP | TCP | 6379 | `10.90.12.0/24` | Allow |
| 230 | Custom TCP | TCP | 16379 | `10.90.11.0/24` | Allow |
| 240 | Custom TCP | TCP | 16379 | `10.90.12.0/24` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules

| Rule No | Type | Protocol | Port | Destination | Allow/Deny |
|---------|------|----------|------|-------------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | HTTP (80) | TCP | 80 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 587 | `0.0.0.0/0` | Allow |
| 130 | Custom TCP | TCP | 27017 | `192.168.248.0/21` | Allow |
| 140 | HTTPS (443) | TCP | 443 | `10.90.0.0/16` | Allow |
| 150 | DNS (TCP) (53) | TCP | 53 | `10.90.0.0/16` | Allow |
| 160 | DNS (UDP) (53) | UDP | 53 | `10.90.0.0/16` | Allow |
| 170 | SSH (22) | TCP | 22 | `20.207.73.82/32` | Allow |
| 180 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 190 | SSH (22) | TCP | 22 | `10.90.11.0/24` | Allow |
| 200 | SSH (22) | TCP | 22 | `10.90.12.0/24` | Allow |
| 210 | Custom TCP | TCP | 6379 | `10.90.11.0/24` | Allow |
| 220 | Custom TCP | TCP | 6379 | `10.90.12.0/24` | Allow |
| 230 | Custom TCP | TCP | 16379 | `10.90.11.0/24` | Allow |
| 240 | Custom TCP | TCP | 16379 | `10.90.12.0/24` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

---

### prod-biddEasy-public-nacl

#### Inbound Rules

| Rule No | Type | Protocol | Port | Source | Allow/Deny |
|---------|------|----------|------|--------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 120 | SSH (22) | TCP | 22 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

#### Outbound Rules

| Rule No | Type | Protocol | Port | Destination | Allow/Deny |
|---------|------|----------|------|-------------|------------|
| 100 | HTTPS (443) | TCP | 443 | `0.0.0.0/0` | Allow |
| 110 | Custom TCP | TCP | 3000 | `10.90.11.0/24` | Allow |
| 120 | Custom TCP | TCP | 3000 | `10.90.12.0/24` | Allow |
| 130 | Custom TCP | TCP | 3002 | `10.90.11.0/24` | Allow |
| 140 | Custom TCP | TCP | 3002 | `10.90.12.0/24` | Allow |
| 150 | Custom TCP | TCP | 3515 | `10.90.11.0/24` | Allow |
| 160 | Custom TCP | TCP | 3515 | `10.90.12.0/24` | Allow |
| 170 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| 180 | SSH (22) | TCP | 22 | `10.90.11.0/24` | Allow |
| 190 | SSH (22) | TCP | 22 | `10.90.12.0/24` | Allow |
| 200 | HTTP (80) | TCP | 80 | `10.90.11.0/24` | Allow |
| 210 | HTTP (80) | TCP | 80 | `10.90.12.0/24` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

---
## VPC Peering Connections

VPC Peering is a networking connection between two VPCs that allows them to communicate privately using private IP addresses. It enables secure data transfer between VPCs without using the internet, VPN, or NAT.

### UAT Environment

| Name | Peering ID | Requester | Accepter | Requester CIDR | Accepter CIDR |
|------|------------|-----------|----------|----------------|---------------|
| `biddprod-to-bidduat` | `pcx-0bfb406f33d3f9d39` | `vpc-07cd70e1627796750` | `vpc-0b1f4a3ecc95086e2` | `10.100.0.0/16` | `10.70.0.0/16` |
| `uat-bidd-vpc-to-atlas-mongodb-uat-cluster` | `pcx-0f9c6969527912b03` | `vpc-0c15c903bb07b2372` | `vpc-0b1f4a3ecc95086e2` | `192.168.248.0/21` | `10.70.0.0/16` |
| `icm-db-nat-to-bidd-uat-vpc` | `pcx-0761b6447a5af5541` | `vpc-30b82559` | `vpc-0b1f4a3ecc95086e2` | `10.0.0.0/16` | `10.70.0.0/16` |

---

## Route53

Amazon Route 53 is used as the central DNS service for the Biddeasy platform. It manages all public domain records and routes user traffic to CloudFront, ALBs, and other AWS services.

### Hosted Zone Details

| Item | Value |
|------|-------|
| **Hosted Zone Name** | `biddeasy.com` |
| **Type** | Public Hosted Zone |
| **Hosted Zone ID** | `Z040560417E3CEE47EV20` |
| **Total Records** | 52 |
| **Managed By** | AWS Route 53 |

This hosted zone is the authoritative DNS for all Biddeasy environments such as Prod, UAT, Staging, India-specific apps, Admin portals, APIs, and static websites.

### UAT-Specific DNS Records

#### uat.biddeasy.com

| Attribute | Value |
|-----------|-------|
| **Record Type** | A (Alias) |
| **Target** | CloudFront |
| **CloudFront Domain** | `d1c7rlm04vujut.cloudfront.net` |
| **Purpose** | Main UAT frontend |

**User Access Flow:**
`uat.biddeasy.com` → Route53 routes traffic to CloudFront → CloudFront forwards requests to ALB for API and S3 static sites for prelogin and postlogin.

---

## AWS Certificate Manager (ACM)

AWS Certificate Manager (ACM) is used to manage SSL/TLS certificates for the Biddeasy platform, enabling HTTPS (secure communication) for all public-facing applications and APIs.

Biddeasy is using one wildcard certificate to secure all subdomains under `biddeasy.com`.

### Certificate Details

| Attribute | Value |
|-----------|-------|
| **Certificate ID** | `65933fc4-4b16-4c5c-90a9-a39f3baa7a8d` |
| **ARN** | `arn:aws:acm:ap-south-1:851725248992:certificate/65933fc4-4b16-4c5c-90a9-a39f3baa7a8d` |
| **Domain Name** | `*.biddeasy.com` |
| **Type** | Amazon Issued |
| **Status** | Issued |
| **Key Algorithm** | RSA 2048 |
| **Signature Algorithm** | SHA-256 with RSA |
| **Renewal Eligibility** | Eligible |
| **In Use** | Yes |

---

## CloudFront

Amazon CloudFront is used as the primary content delivery and entry layer for the Biddeasy platform.

It sits in front of:
- Static websites hosted on S3
- Dynamic backend APIs hosted behind Application Load Balancers (ALB)

### UAT Environment

| Distribution ID | Domain |
|-----------------|--------|
| `E2KZCJ8FDJGAEJ` | `uat.biddeasy.com` |
| `E25E28HZBXY6BS` | `uatbonds.biddeasy.com` |
| `E2I3UFIXUYD3T2` | `uatlearn.biddeasy.com` |
| `EKTYH98S9H6JW` | `adminuat.biddeasy.com` |

### Staging Environment

| Distribution ID | Domain |
|-----------------|--------|
| `E3QFC7XE1FHX6K` | `india.biddeasy.com` |
| `E32VHFUFBBBLAY` | `indiabonds.biddeasy.com` |
| `E2F788R35RPUKJ` | `indiaadmin.biddeasy.com` |
| `E33TGKD4W8CV6` | `indialearn.biddeasy.com` |

### Production Environment

| Distribution ID | Domain |
|-----------------|--------|
| `E1K8UQQSESDI1N` | `www.biddeasy.com` |
| `E1U0QPF8VYZ1EN` | `biddeasy.com` |
| `E7ZJAS2B8RUZF` | `assets.biddeasy.com` |
| `EDSJ251MAIEIU` | `admin.biddeasy.com` |
| `E2UHCF9487VOYA` | `bonds.biddeasy.com` |
| `E3RL25CLD3BB45` | `learn.biddeasy.com` |
| `E1QYCUCJUDH84J` | `marketing.biddeasy.com` |

---

## Deployment Steps

For detailed deployment procedures, refer to the deployment documentation:

[Biddeasy Deployment Guide](https://docs.google.com/document/d/1mI7rtrBRfdnXjAsPhYxab5b_PSiQaJKeoLYakEdSiVY/edit?usp=sharing)

---



