# Biddeasy Infrastructure Documentation

## Table of Contents

### 1. [VPC (Virtual Private Cloud)](#vpc-virtual-private-cloud)

### 2. [Routing Tables](#routing-tables)
   - [UAT Environment](#uat-environment)
   - [Staging Environment](#staging-environment)
   - [Production Environment](#production-environment)

### 3. [NAT Gateway](#nat-network-address-translation)

### 4. [Internet Gateway](#igw-internet-gateway)

### 5. [VPC Endpoints](#vpc-endpoints)
   - [UAT Environment](#uat-environment-1)
   - [Staging Environment](#staging-environment-1)
   - [Production Environment](#production-environment-1)

### 6. [Security Groups](#security-groups)
   - [UAT Environment](#uat-environment-2)
     - [uat-biddEasy-app-be-sg](#uat-biddeasy-app-be-sg)
     - [uat-biddEasy-alb-sg](#uat-biddeasy-alb-sg)
     - [uat-biddEasy-app-fe-sg](#uat-biddeasy-app-fe-sg)
     - [uat-biddEasy-vpc-endpoint-sg](#uat-biddeasy-vpc-endpoint-sg)
   - [Staging Environment](#staging-environment-2)
     - [staging-biddEasy-app-fe-sg](#staging-biddeasy-app-fe-sg)
     - [staging-biddEasy-vpc-endpoint-sg](#staging-biddeasy-vpc-endpoint-sg)
     - [staging-biddEasy-bastion-sg](#staging-biddeasy-bastion-sg)
     - [staging-biddEasy-alb-sg](#staging-biddeasy-alb-sg)
   - [Production Environment](#production-environment-2)
     - [prod-biddEasy-app-fe-sg](#prod-biddeasy-app-fe-sg)
     - [prod-biddEasy-bastion-sg](#prod-biddeasy-bastion-sg)
     - [prod-biddEasy-redis-sg](#prod-biddeasy-redis-sg)
     - [prod-biddEasy-alb-sg](#prod-biddeasy-alb-sg)
     - [prod-biddEasy-app-be-sg](#prod-biddeasy-app-be-sg)
     - [prod-biddEasy-vpc-endpoint-sg](#prod-biddeasy-vpc-endpoint-sg)

### 7. [NACL (Network Access Control List)](#nacl-network-access-control-list)
   - [UAT Environment](#uat-environment-3)
     - [uat-biddEasy-private-nacl](#uat-biddeasy-private-nacl)
     - [uat-biddEasy-public-nacl](#uat-biddeasy-public-nacl)
   - [Staging Environment](#staging-environment-3)
     - [staging-biddEasy-private-nacl](#staging-biddeasy-private-nacl)
     - [staging-biddEasy-public-nacl](#staging-biddeasy-public-nacl)
   - [Production Environment](#production-environment-3)
     - [prod-biddEasy-private-nacl](#prod-biddeasy-private-nacl)
     - [prod-biddEasy-public-nacl](#prod-biddeasy-public-nacl)

### 8. [VPC Peering Connections](#vpc-peering-connections)

### 9. [Route53](#route53)

### 10. [AWS Certificate Manager (ACM)](#aws-certificate-manager-acm)

### 11. [CloudFront](#cloudfront)
   - [UAT Environment](#uat-environment-4)
   - [Staging Environment](#staging-environment-4)
   - [Production Environment](#production-environment-4)

### 12. [Deployment Steps](#deployment-steps)

---

## VPC (Virtual Private Cloud)

A VPC (Virtual Private Cloud) is a logically isolated private network in AWS where you can launch and manage resources securely. It allows you to define your own IP range, subnets, routing, and security controls.

### VPC Configuration

| S/N | VPC Name | VPC ID | IPv4 CIDR |
|-----|----------|--------|-----------|
| 1 | `bidd-uat-vpc` | `vpc-0b1f4a3ecc95086e2` | `10.70.0.0/16` |
| 2 | `bidd-staging-vpc` | `vpc-01aa775625bdbc866` | `10.80.0.0/16` |
| 3 | `bidd-prod-vpc` | `vpc-0ba146c7ffff6ddf4` | `10.90.0.0/16` |

---
