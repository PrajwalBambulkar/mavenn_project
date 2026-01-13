# BIDD UAT Deployment Pipeline – Server-Wise Complete Documentation

---

## 🔹 Quick Overview

This pipeline involves **3 servers**:

1. **Jenkins Build Server** – Where code is built and security-scanned  
2. **AWS EC2 (UAT App Server)** – Where the application runs  
3. **Mail Server (Jenkins Agent)** – Sends security alerts  

**High-Level Flow**

Git (App Repo + prodConf Repo)
↓
Jenkins Build Server
↓
Trivy Security Scan
↓
Build (Next.js)
↓
Deploy to UAT EC2 (Private IP)
↓
PM2 Reload / Start


---

# 🖥️ SERVER 1: JENKINS BUILD SERVER

## Server Details

| Item | Value |
|----|----|
| Hostname | Jenkins Server |
| Node Label | `biddscripts` |
| OS | Ubuntu |
| Primary User | `ubuntu` |

---

## Prerequisites Required on Jenkins Server

### 1. Node.js & Package Manager

- **Node Version**: `v20.11.1`
- **Installed via**: NVM
- **Install Path**:
/home/ubuntu/.nvm/versions/node/v20.11.1/

**Binaries**
Node: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/node
Yarn: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn


**Verification**
```bash
/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node --version
/home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn --version
