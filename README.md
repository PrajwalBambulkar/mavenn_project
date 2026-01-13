# **BIDD Handler UAT Deployment Pipeline \- Complete Documentation**

## **📋 Quick Overview**

This pipeline deploys the **BIDD Handler (Backend)** to UAT environment with flexible deployment options.

**Pipeline involves 3 components:**

1. **Jenkins Build Server** \- Where code is synced and scanned  
2. **AWS EC2 (UAT Servers)** \- Multiple backend servers where API runs  
3. **Mail Server** \- Sends security alerts (uses Jenkins server's sendmail)

### **Pipeline Flow**

```
Git Repositories (Handler + Communications + Config)
↓
Jenkins Build Server (Sync & Scan)
↓
Trivy Security Scan
↓
Deploy to Multiple UAT API Servers (Sequential)
↓
PM2 Reload on Each Server
```

### **🖥️ SERVER 1: JENKINS BUILD SERVER**

### **Server Details**

* **Node Label**: `biddscripts`  
* **Operating System**: Ubuntu  
* **Primary User**: `ubuntu`  
* **Purpose**: Code sync, security scanning (NO building required)

### **Prerequisites Required on Jenkins Server**

#### **1\. Node.js & NPM**

```shell
# Node.js version: v20.11.1
# Installation path: /home/ubuntu/.nvm/versions/node/v20.11.1/

Node Binary: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/node
NPM Binary: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/npm
```

**To verify:**

```shell
/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node --version
# Should output: v20.11.1

/home/ubuntu/.nvm/versions/node/v20.11.1/bin/npm --version
# Should output: 10.x.x
```

#### **2\. Git with SSH Keys**

```shell
git --version
# Should output: git version 2.x.x
```

**SSH Keys Location:**

```
/home/ubuntu/.ssh/id_rsa (private key)
/home/ubuntu/.ssh/id_rsa.pub (public key)
```

**GitHub SSH Access Required For:**

* `git@github.com:incredmoney/IncredMoney_handler.git`  
* `git@github.com:incredmoney/app.bidd.communications.git`  
* `git@github.com:incredmoney/prodConf.git`

#### **3\. Trivy (Security Scanner)**

```shell
trivy --version
# Should output: Version: 0.x.x
```

**Installation:**

```shell
wget https://github.com/aquasecurity/trivy/releases/download/v0.48.0/trivy_0.48.0_Linux-64bit.deb
sudo dpkg -i trivy_0.48.0_Linux-64bit.deb
```

#### **4\. AWS CLI**

```shell
aws --version
# Should output: aws-cli/2.x.x

aws configure list
# Should show access key and region: ap-south-1
```

#### **5\. SSH Client**

```shell
ssh -V
# Should output: OpenSSH_8.x
```

**SSH Keys for UAT Servers:**

```
Location: /home/ubuntu/.ssh/id_rsa
Public key must be in: biddbeuat@UAT-SERVERS:~/.ssh/authorized_keys
```

#### **6\. Sendmail**

```shell
sendmail -v
# Should show version info
```

### **Directory Structure on Jenkins Build Server**

```
/home/ubuntu/
├── .nvm/
│   └── versions/
│       └── node/
│           └── v20.11.1/
│               └── bin/
│                   ├── node          ← Node.js binary
│                   └── npm           ← NPM binary
│
└── workspace/
    └── <JOB_NAME>/
        ├── IncredMoney_handler/      ← Main API repository
        │   ├── conf/
        │   │   ├── conf.json         ← Handler configuration
        │   │   └── products.json     ← Products configuration
        │   ├── app.bidd.communications/  ← Git submodule
        │   │   └── (communication module code)
        │   ├── node_modules/         ← After npm install
        │   ├── package.json
        │   └── (API source code)
        │
        └── handler_trivy_report.txt  ← Trivy scan output
```

---

## **Where Repositories Are Cloned**

### **Repository 1: Handler(Main Repository)**

```shell
Repository URL: git@github.com:incredmoney/IncredMoney_handler.git
Clone Location: ${WORKSPACE}/IncredMoney_handler
Full Path: /home/ubuntu/workspace/<JOB_NAME>/IncredMoney_handler
Branch: develop
Cloned By: Stage 2 (Code Sync + NPM + Trivy)

# Clone command:
git clone -b develop git@github.com:incredmoney/IncredMoney_handler.git \
    ${WORKSPACE}/IncredMoney_handler
```

### **Repository 2: Communications Module (Git Submodule)**

```shell
Repository URL: git@github.com:incredmoney/app.bidd.communications.git
Clone Location: ${WORKSPACE}/IncredMoney_handler/app.bidd.communications
Full Path: /home/ubuntu/workspace/<JOB_NAME>/IncredMoney_handler/app.bidd.communications
Branch: develop
Type: Git Submodule
Cloned By: Stage 2 (as part of Handler repo submodule init)

# Submodule command:
cd IncredMoney_handler
git submodule add -b develop \
    git@github.com:incredmoney/app.bidd.communications.git \
    app.bidd.communications
```

### **Repository 3: Configuration Repository (prodConf)**

```shell
Repository URL: git@github.com:incredmoney/prodConf.git
Clone Location: Cloned on UAT servers (not on Jenkins)
Branch: master
Cloned By: Stage 4 (Deploy to App Servers) - only if WITH_CONF or WITH_PRODUCTS_CONF is true

# This repo is cloned on each UAT server:
Location on UAT: /home/biddbeuat/prodConf/

Contains:
- /home/biddbeuat/prodConf/biddEasy/uat/handler/conf.json
- /home/biddbeuat/prodConf/biddEasy/uat/handler/products.json
```

---

## ** Pipeline Parameters (Deployment Modes)**

The pipeline has **3 boolean parameters** that control what gets deployed:

| Parameter | Default | Description | What It Does |
| ----- | ----- | ----- | ----- |
| `WITH_CONF` | `false` | Deploy Handler configuration (conf.json) | Copies `conf.json` from prodConf to Handler |
| `WITH_PRODUCTS_CONF` | `false` | Deploy Handler configuration (products.json) | Copies `products.json` from prodConf to Handler |
| `WITH_COMMS` | `false` | Deploy Handler communications module | Updates communications submodule |

### **Deployment Scenarios**

#### **Scenario 1: Code-Only Deployment (All parameters \= false)**

```
✅ Handler API code updated
❌ Configuration files NOT updated
❌ Communications module NOT updated
```

#### **Scenario 2: Code \+ Configuration (WITH\_CONF \= true)**

```
✅ Handler API code updated
✅ conf.json updated from prodConf
❌ products.json NOT updated
❌ Communications module NOT updated
```

#### **Scenario 3: Full Deployment (All parameters \= true)**

```
✅ Handler API code updated
✅ conf.json updated
✅ products.json updated
✅ Communications module updated
```

---

## **🔄 PIPELINE STAGES**

### **Stage 1: Validate Parameters**

```
stage('Validate Parameters') {
    steps {
        echo "Deployment Mode: WITH_CONF=${params.WITH_CONF}, WITH_PRODUCTS_CONF=${params.WITH_PRODUCTS_CONF}, WITH_COMMS=${params.WITH_COMMS}"
    }
}
```

**Purpose:** Display which deployment mode is selected

**Output Example:**

```
Deployment Mode: WITH_CONF=true, WITH_PRODUCTS_CONF=false, WITH_COMMS=true
```

---

### **Stage 2: Code Sync \+ NPM \+ Trivy**

**Server:** Jenkins Build Server  
 **User:** ubuntu  
 **Working Directory:** `${WORKSPACE}/IncredMoney_handler`

#### **Step 2.1: Clone/Sync Handler Repository**

```shell
# If directory doesn't exist, clone it
if [ ! -d "${WORKSPACE}/IncredMoney_handler" ]; then
    git clone -b develop git@github.com:incredmoney/IncredMoney_handler.git \
        ${WORKSPACE}/IncredMoney_handler
fi

# If exists, sync it
cd ${WORKSPACE}/IncredMoney_handler
git stash
git fetch origin develop
git rebase origin/develop
git stash pop || true
```

#### **Step 2.2: Initialize Communications Submodule**

```shell
# Ensure submodule structure exists
if [ ! -d "app.bidd.communications" ]; then
    git rm -rf app.icm.communications || true
    git submodule add -b develop \
        git@github.com:incredmoney/app.bidd.communications.git \
        app.bidd.communications || true
fi
```

**Why this step**

* Removes old submodule `app.icm.communications` (if exists)  
* Adds new submodule `app.bidd.communications`  
* Uses develop branch

#### **Step 2.3: Install Dependencies**

```shell
npm install
```

**Creates:** `node_modules/` directory in `${WORKSPACE}/IncredMoney_handler/`

#### **Step 2.4: Trivy Security Scan**

```shell
trivy fs . \
    --severity LOW,MEDIUM,HIGH,CRITICAL \
    --exit-code 0 \
    --format table \
    --output ${WORKSPACE}/handler_trivy_report.txt
```

**Trivy Configuration:**

| Parameter | Value | Meaning |
| ----- | ----- | ----- |
| `fs .` | Current directory | Scan entire Handler repo |
| `--severity` | LOW,MEDIUM,HIGH,CRITICAL | All severity levels |
| `--exit-code 0` | **Does NOT fail build** | Only logs vulnerabilities |
| `--format table` | Table format | Human-readable output |
| `--output` | `${WORKSPACE}/handler_trivy_report.txt` | Save to file |

**⚠️ Important Difference from UI Pipeline:**

* UI Pipeline: `--exit-code 1` (FAILS build if vulnerabilities found)  
* Handler Pipeline: `--exit-code 0` (DOES NOT fail build, only sends email alert)

### **Where Trivy Report Is Generated**

```shell
Report Location: ${WORKSPACE}/handler_trivy_report.txt
Full Path: /home/ubuntu/workspace/<JOB_NAME>/handler_trivy_report.txt

Generated By: Stage 2 (Code Sync + NPM + Trivy)
Used By: Post-build failure handler (email attachment)
Exit Code: 0 (does not fail pipeline)
```

---

### **Stage 3: Fetch Server IPs**

**Server:** Jenkins Build Server  
 **User:** ubuntu

```shell
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=bidd-be-uat" \
              "Name=instance-state-name,Values=running" \
    --query "Reservations[].Instances[].PrivateIpAddress" \
    --output text
```

**Output Example:**

```
10.0.2.10 10.0.2.11 10.0.2.12
```

**Stored in variable:** `${APP_SERVER_IPS}`

**Key Point:** This fetches **MULTIPLE** API server IPs (unlike UI pipeline which has only one server)

---

### **Stage 4: Deploy to App Servers (Sequential)**

**Server:** UAT Servers (Multiple)  
 **User:** biddbeuat  
 **Method:** SSH from Jenkins → Git pull on each server

**Deployment Strategy:**

```
For each IP in APP_SERVER_IPS:
    SSH to server
    Git pull Handler code
    Update configurations (if enabled)
    Update communications (if enabled)
    PM2 reload
Next IP
```

#### **Deployment Loop**

```shell
for ip in env.APP_SERVER_IPS.tokenize(' '); do
    echo "Deploying to ${ip}"
    ssh -o StrictHostKeyChecking=no biddbeuat@${ip} << 'EOF'
    
    # Deployment steps here...
    
EOF
done
```

#### **Step 4.1: Sync Handler Code on UAT Server**

```shell
cd /home/biddbeuat/IncredMoney_handler
git stash
git fetch origin develop
git rebase origin/develop
git stash pop || true

npm install
```

**Important:** This happens **ON THE UAT SERVER**, not on Jenkins\!

#### **Step 4.2: Update conf.json (If WITH\_CONF \= true)**

```shell
if [ "${params.WITH_CONF}" = "true" ]; then
    echo "Updating conf.json Configuration..."
    
    # Clone/sync prodConf repo on UAT server
    cd /home/biddbeuat
    [ ! -d prodConf ] && git clone -b master git@github.com:incredmoney/prodConf.git
    cd prodConf
    git stash
    git fetch origin master
    git rebase origin/master
    git stash pop || true
    
    # Copy configuration file
    cp /home/biddbeuat/prodConf/biddEasy/uat/handler/conf.json \
       /home/biddbeuat/IncredMoney_handler/conf/conf.json
fi
```

**File Copy:**

```
Source: /home/biddbeuat/prodConf/biddEasy/uat/handler/conf.json
Destination: /home/biddbeuat/IncredMoney_handler/conf/conf.json
```

#### **Step 4.3: Update products.json (If WITH\_PRODUCTS\_CONF \= true)**

```shell
if [ "${params.WITH_PRODUCTS_CONF}" = "true" ]; then
    echo "Updating products.json Configuration..."
    
    # Clone/sync prodConf repo on UAT server
    cd /home/biddbeuat
    [ ! -d prodConf ] && git clone -b master git@github.com:incredmoney/prodConf.git
    cd prodConf
    git stash
    git fetch origin master
    git rebase origin/master
    git stash pop || true
    
    # Copy configuration file
    cp /home/biddbeuat/prodConf/biddEasy/uat/handler/products.json \
       /home/biddbeuat/IncredMoney_handler/conf/products.json
fi
```

**File Copy:**

```
Source: /home/biddbeuat/prodConf/biddEasy/uat/handler/products.json
Destination: /home/biddbeuat/IncredMoney_handler/conf/products.json
```

#### **Step 4.4: Update Communications Module (If WITH\_COMMS \= true)**

```shell
if [ "${params.WITH_COMMS}" = "true" ]; then
    echo "Updating Communications Module..."
    
    cd /home/biddbeuat/IncredMoney_handler/app.bidd.communications
    git stash
    git fetch origin develop
    git rebase origin/develop
    git stash pop || true
fi
```

**Submodule Location:**

```
/home/biddbeuat/IncredMoney_handler/app.bidd.communications/
```

#### **Step 4.5: Reload PM2 Process**

```shell
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload handler
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
```

**PM2 App Name:** `handler`

---

## **🖥️ SERVER 2: UAT API SERVERS (AWS EC2 \- Multiple Instances)**

### **Server Details**

* **Server Type**: AWS EC2 Instances (Multiple)  
* **AWS Region**: `ap-south-1` (Mumbai)  
* **Instance Tag**: `Name=bidd-be-uat`  
* **IP Type**: Private IPs (Dynamic \- fetched via AWS CLI)  
* **Operating System**: Ubuntu  
* **User**: `biddbeuat`

### **How Server IPs Are Discovered**

```shell
aws ec2 describe-instances \
    --region ap-south-1 \
    --filters 'Name=tag:Name,Values=bidd-be-uat' \
              'Name=instance-state-name,Values=running' \
    --query 'Reservations[].Instances[].PrivateIpAddress' \
    --output text

# Example output: 10.0.2.10 10.0.2.11 10.0.2.12
# Stored in: ${APP_SERVER_IPS}
```

### **User Account on UAT Servers**

```shell
Username: biddbeuat
Home Directory: /home/biddbeuat
Shell: /bin/bash

# SSH Access
From: Jenkins Build Server (ubuntu user)
To: biddbeuat@<EACH_UAT_IP>
Authentication: SSH key-based (passwordless)
```

### **Prerequisites on Each UAT Server**

#### **1\. Node.js (v20.19.6)**

```shell
# Installation path: /home/biddbeuat/.nvm/versions/node/v20.19.6/

Node Binary: /home/biddbeuat/.nvm/versions/node/v20.19.6/bin/node
NPM Binary: /home/biddbeuat/.nvm/versions/node/v20.19.6/bin/npm
PM2 Binary: /home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2
```

**To verify:**

```shell
ssh biddbeuat@<UAT_IP>
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/node --version
# Should output: v20.19.6
```

#### **2\. PM2 (Process Manager)**

```shell
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 --version
# Should output: 5.x.x
```

#### **3\. Git with SSH Keys**

```shell
git --version
# Should output: git version 2.x.x
```

**SSH Keys Location:**

```
/home/biddbeuat/.ssh/id_rsa (private key)
/home/biddbeuat/.ssh/id_rsa.pub (public key)
```

**GitHub Access Required:**

* Handler repo: `git@github.com:incredmoney/IncredMoney_handler.git`  
* Communications repo: `git@github.com:incredmoney/app.bidd.communications.git`  
* Config repo: `git@github.com:incredmoney/prodConf.git`

#### **4\. SSH Server (for Jenkins access)**

```shell
sudo systemctl status sshd
# Should show: active (running)

# Authorized keys
/home/biddbeuat/.ssh/authorized_keys
# Must contain public key from Jenkins (ubuntu@jenkins)
```

### **Directory Structure on UAT Servers**

```
/home/biddbeuat/
├── .nvm/
│   └── versions/
│       └── node/
│           └── v20.19.6/
│               └── bin/
│                   ├── node          ← Node.js binary
│                   ├── npm           ← NPM binary
│                   └── pm2           ← PM2 process manager
│
├── .pm2/
│   ├── logs/
│   │   ├── handler-out.log          ← Application stdout logs
│   │   └── handler-error.log        ← Application error logs
│   └── dump.pm2                     ← PM2 saved process list
│
├── prodConf/                        ← Config repository (cloned if WITH_CONF or WITH_PRODUCTS_CONF)
│   └── biddEasy/
│       └── uat/
│           └── handler/
│               ├── conf.json        ← Handler configuration
│               └── products.json    ← Products configuration
│
└── IncredMoney_handler/             ← Main API application
    ├── conf/
    │   ├── conf.json                ← Runtime configuration (copied from prodConf)
    │   └── products.json            ← Products configuration (copied from prodConf)
    ├── app.bidd.communications/     ← Git submodule
    │   └── (communication module code)
    ├── node_modules/                ← NPM dependencies
    ├── package.json
    ├── index.js                     ← Main entry point
    └── (other API source files)
```

---

##  **PM2 Process Management on UAT Servers**

### **PM2 Application Configuration**

```shell
Application Name: handler
Process Manager: PM2
Working Directory: /home/biddbeuat/IncredMoney_handler
Entry Point: index.js (or as defined in package.json)
```

### **PM2 Commands Used in Pipeline**

```shell
# Reload application (zero-downtime restart)
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload handler

# Save PM2 configuration
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
```

### **Manual PM2 Operations**

```shell
# SSH to any UAT server
ssh biddbeuat@<UAT_IP>

# Check if app is running
pm2 list
# Should show: handler | online | ...

# View logs
pm2 logs handler

# View last 100 lines
pm2 logs handler --lines 100

# Restart app
pm2 restart handler

# Stop app
pm2 stop handler

# Delete app
pm2 delete handler

# Start app manually
pm2 start index.js --name handler

# Monitor CPU/Memory
pm2 monit

# View detailed info
pm2 show handler
```

---

## **📧 SERVER 3: MAIL SERVER (Jenkins Agent)**

### **Mail Configuration**

```shell
Mail Server: Uses Jenkins Build Server's sendmail
From Address: Jenkins <connect@biddeasy.com>
To Address: rajat.sah@incredmoney.com
SMTP Configuration: Configured on Jenkins server
```

### **When Emails Are Sent**

```
Trigger: Pipeline failure (any stage failure)
Condition: handler_trivy_report.txt exists AND contains vulnerabilities
Subject: [SECURITY ALERT] Vulnerabilities Found - ${JOB_NAME} #${BUILD_NUMBER}
Attachment: handler_trivy_report.txt
```

### **Email Alert Logic**

```
post {
    failure {
        script {
            // Check if Trivy report exists and has vulnerabilities
            def hasHandlerVuls = checkVuls("${WORKSPACE}/handler_trivy_report.txt")
            
            if (hasHandlerVuls) {
                // Send security alert email with attachment
            } else {
                echo "No security vulnerabilities detected"
            }
        }
    }
}
```

**Vulnerability Detection:**

```shell
# Checks if report contains vulnerability counts
grep -E 'Total: [1-9]|[0-9]+ (CRITICAL|HIGH|MEDIUM|LOW)' handler_trivy_report.txt
```

### **Email Content**

```html
From: Jenkins <connect@biddeasy.com>
To: rajat.sah@incredmoney.com
Subject: [SECURITY ALERT] Vulnerabilities Found - <JOB_NAME> #<BUILD_NUMBER>

HTML Body:
- Job Name: <JOB_NAME>
- Build Number: #<BUILD_NUMBER>
- Environment: UAT
- Findings: Handler
- Message: "Jenkins blocked deployment"

Attachment: handler_trivy_report.txt
```

---

## **🔄 COMPLETE DATA FLOW**

### **Visual Flow Diagram**

```
┌─────────────────────────────────────────────────────────────────┐
│ JENKINS BUILD SERVER (ubuntu user)                              │
│ /home/ubuntu/workspace/<JOB_NAME>/                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ STAGE 1: Validate Parameters                                    │
│ └─ Display: WITH_CONF, WITH_PRODUCTS_CONF, WITH_COMMS          │
│                                                                  │
│ STAGE 2: Code Sync + NPM + Trivy                               │
│ ├─ Clone/Sync Handler repo                                      │
│ │  Location: ${WORKSPACE}/IncredMoney_handler/                 │
│ ├─ Initialize Communications submodule                          │
│ │  Location: ${WORKSPACE}/IncredMoney_handler/                 │
│ │             app.bidd.communications/                          │
│ ├─ NPM install                                                  │
│ │  Creates: node_modules/                                       │
│ └─ Trivy security scan                                          │
│    Output: ${WORKSPACE}/handler_trivy_report.txt               │
│    Exit Code: 0 (does not fail build)                          │
│                                                                  │
│ STAGE 3: Fetch  Server IPs                                  │
│ └─ AWS CLI: Get IPs of bidd-be-uat instances                   │
│    Result: APP_SERVER_IPS = "10.0.2.10 10.0.2.11 10.0.2.12"    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Sequential SSH Deployment
                     │ (No file copy - Git pull on each server)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT SERVER 1: 10.0.2.10 (biddbeuat user)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ STAGE 4: Deploy to App Servers                                 │
│ ├─ Git pull Handler code                                        │
│ │  Location: /home/biddbeuat/IncredMoney_handler/             │
│ │  Branch: develop                                              │
│ ├─ NPM install                                                  │
│ │                                                               │
│ ├─ [IF WITH_CONF = true]                                       │
│ │  Clone/sync: /home/biddbeuat/prodConf/                       │
│ │  Copy: prodConf/.../conf.json → Handler/conf/conf.json      │
│ │                                                               │
│ ├─ [IF WITH_PRODUCTS_CONF = true]                              │
│ │  Copy: prodConf/.../products.json → Handler/conf/products.json│
│ │                                                               │
│ ├─ [IF WITH_COMMS = true]                                      │
│ │  Git pull communications submodule                            │
│ │  Location: /home/biddbeuat/IncredMoney_handler/             │
│ │             app.bidd.communications/                          │
│ │                                                               │
│ └─ PM2 reload handler                                           │
│    PM2 save                                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Deploy to next server
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT SERVER 2: 10.0.2.11 (biddbeuat user)                   │
│ (Same steps as Server 1)                                        │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Deploy to next server
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT SERVER 3: 10.0.2.12 (biddbeuat user)                   │
│ (Same steps as Server 1)                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MAIL SERVER (if build fails)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ POST-BUILD: Failure Handler                                     │
│ ├─ Check: handler_trivy_report.txt exists?                      │
│ ├─ Check: Contains vulnerabilities?                             │
│ └─ If yes, send email:                                          │
│    From: Jenkins <connect@biddeasy.com>                         │
│    To: rajat.sah@incredmoney.com                                │
│    Subject: [SECURITY ALERT] Vulnerabilities Found              │
│    Attachment: handler_trivy_report.txt                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## **PATH REFERENCE TABLE**

### **Jenkins Build Server Paths**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| Handler Repository | `${WORKSPACE}/IncredMoney_handler/` | Handler source code |
| Communications Submodule | `${WORKSPACE}/IncredMoney_handler/app.bidd.communications/` | Communications module |
| Trivy Report | `${WORKSPACE}/handler_trivy_report.txt` | Security scan results |
| Node.js Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node` | Node runtime |
| NPM Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/npm` | Package manager |
| SSH Private Key | `/home/ubuntu/.ssh/id_rsa` | For GitHub and UAT access |

### **UAT Server Paths (Each Server)**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| Handler Application | `/home/biddbeuat/IncredMoney_handler/` | Running application |
| Communications Submodule | `/home/biddbeuat/IncredMoney_handler/app.bidd.communications/` | Communications module |
| Configuration Repository | `/home/biddbeuat/prodConf/` | Environment configs |
| conf.json Source | `/home/biddbeuat/prodConf/biddEasy/uat/handler/conf.json` | Handler config source |
| conf.json Destination | `/home/biddbeuat/IncredMoney_handler/conf/conf.json` | Handler config (runtime) |
| products.json Source | \`/home/biddbe |  |
