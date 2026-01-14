# BIDD – Jenkins Pipelines


## The BIDD project uses three separate Jenkins pipelines to manage build and deployment for different components:

- Frontend (FE) Pipeline

- Backend API Pipeline

- Backend Handler Pipeline

# **BIDD FE Pipeline Complete Documentation**
https://docs.google.com/document/d/1-bAJlUfgethEJOCcxlf_Jyp4SVjsZm9DPRJjzXbXmv8/edit?usp=sharing
## **Quick Overview**

This pipeline involves **3 servers**:	

1. **Jenkins Build Server** \- Where code is built and scanned  
2. **AWS EC2 (UAT App Server)** \- Where application runs  
3. **Mail Server** \- Sends security alerts (uses Jenkins Agent server)

Git (App Repo \+ prodConf Repo)  
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

## **🖥️ SERVER 1: JENKINS BUILD SERVER**

### **Server Details**

* **Hostname**:   
* **Node Label**: `biddscripts`  
* **Operating System**: Ubuntu  
* **Primary User**: `ubuntu`

### **Prerequisites Required on This Server**

#### **1\. Node.js & Package Manager**

```shell
# Node.js version: v20.11.1
# Installation path: /home/ubuntu/.nvm/versions/node/v20.11.1/
# Installed via: NVM (Node Version Manager)

Node Binary: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/node
Yarn Binary: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn
```

**To verify**:

```shell
/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node --version
# Should output: v20.11.1

/home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn --version
# Should output: 1.x.x or 3.x.x
```

#### **2\. Git**

```shell
# Required for: Cloning repositories
# Must have SSH keys configured for GitHub
git --version
# Should output: git version 2.x.x
```

**SSH Keys Location**:

```
/home/ubuntu/.ssh/id_rsa (private key)
/home/ubuntu/.ssh/id_rsa.pub (public key)
```

**GitHub SSH Access Required For**:

* `git@github.com:incredmoney/prodConf.git`  
* `git@github.com:incredmoney/web.bidd.application.git`

#### **3\. Trivy (Security Scanner)**

```shell
# Required for: Vulnerability scanning
trivy --version
# Should output: Version: 0.x.x
```

**Installation** (if not installed):

```shell
wget https://github.com/aquasecurity/trivy/releases/download/v0.48.0/trivy_0.48.0_Linux-64bit.deb
sudo dpkg -i trivy_0.48.0_Linux-64bit.deb
```

#### **4\. AWS CLI**

```shell
# Required for: Fetching EC2 instance IP
aws --version
# Should output: aws-cli/2.x.x

# Must be configured with credentials
aws configure list
# Should show access key and region
```



#### **5\. SSH Client**

```shell
# Required for: Deploying to UAT server
ssh -V
# Should output: OpenSSH_8.x

# Required for: Copying files to UAT server
scp --version
# Should output: OpenSSH_8.x
```

**SSH Keys for UAT Server**:

```
Location: /home/ubuntu/.ssh/
Private Key: /home/ubuntu/.ssh/id_rsa
Public Key must be added to: biddfeuat@UAT-SERVER:~/.ssh/authorized_keys
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
│                   └── yarn          ← Yarn binary
│
├── workspace/
│   └── bidd_ui_uat/
│       └── web.bidd.application/     ← BUILD_PATH (Main workspace)
│           ├── .next/                ← Generated after build
│           ├── public/               ← Static files
│           ├── node_modules/         ← Dependencies (after yarn install)
│           ├── package.json          ← Dependencies list
│           ├── yarn.lock             ← Locked versions
│           ├── next.config.ts        ← Copied from prodConf
│           └── src/                  ← Source code
│
└── prodConf/                         ← PRODCONF_PATH (Config repo)
    └── biddEasy/
        └── uat/
            └── bidd2.0/
                └── next.config.ts    ← UAT environment config
```



#### **Repository 1: Configuration Repository (prodConf)**

### **Stage 1: Setup Configuration**

```shell
stages {
        stage('Setup Configuration') {
            steps {
                script {
                    echo "=== Syncing prodConf Repository ==="
                    sh """
                        if [ ! -d "${env.PRODCONF_PATH}" ]; then
                            git clone ${env.CONF_REPO_URL} ${env.PRODCONF_PATH}
                        fi
                        cd ${env.PRODCONF_PATH}
                        git stash
                        git fetch origin master
                        git rebase origin/master
                        git stash pop || true
                    """
                }
            }
        }
```

### 

**Purpose**: Sync the latest configuration files from the prodConf repository

**Process**:

1. Checks if `prodConf` directory exists  
2. If not exists, clones the repository (master branch)  
3. If exists:  
   * Stashes any local changes  
   * Fetches latest changes from origin/master  
   * Rebases on origin/master  
   * Restores stashed changes (if any)

**Why This Matters**: Ensures you always have the latest configuration files (like `next.config.ts`) before building.

```shell
Repository URL: git@github.com:incredmoney/prodConf.git
Clone Location: /home/ubuntu/prodConf
Branch: master
Cloned By: Stage 1 (Setup Configuration)

# Clone command used:
git clone git@github.com:incredmoney/prodConf.git /home/ubuntu/prodConf
```

**What's inside**:

```
/home/ubuntu/prodConf/
└── biddEasy/
    ├── uat/
    │   └── bidd2.0/
    │       └── next.config.ts        ← Used in Stage 3
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

#### 

## **Stage 2: Initialize & Fetch App IP**

### **Purpose**

* Dynamically fetch **current UAT EC2 private IP**

### **How It Works**

* AWS CLI query using:  
  * EC2 tag: `Name=bidd-fe-uat`  
  * Instance state: `running`
```shell
aws ec2 describe-instances \\  
\--filters Name=tag:Name,Values=bidd-fe-uat \\  
Name=instance-state-name,Values=running
```
### **Safety Check**

* Pipeline **fails immediately** if:  
  * No IP found  
  * Instance is stopped  
  * 

### **Stage 3: Build & Security Scan**

**Purpose**: Build the Next.js application and scan for security vulnerabilities

#### 

#### **Repository 2: Application Source Code**

```shell
Repository URL: git@github.com:incredmoney/web.bidd.application.git
Clone Location: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
Branch: uat (configurable via pipeline parameter)
Cloned By: Stage 3 (Build & Security Scan)

# Clone command used:
git clone -b uat git@github.com:incredmoney/web.bidd.application.git \
    /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
# Clone if not exists, otherwise sync
   
   cd ${BUILD_PATH}
   git stash
   git fetch origin uat
   git rebase origin/uat
   git stash pop || true
```

**Configuration Injection**:

bash

```shell
   cp ${PRODCONF_PATH}/biddEasy/uat/bidd2.0/next.config.ts .
```

* Copies environment-specific Next.js configuration  
* Overwrites default config with UAT settings

bash

```shell
   yarn install
```

* Installs all npm packages from `package.json`


### **Trivy Report**

```shell
Report Location: ${WORKSPACE}/trivy_report.txt
Full Path: /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt

# Generated By: Stage 3 (Trivy security scan)
# Used By: Post-build failure handler (email attachment)
# Cleaned Up: Post-build always handler

# Command that generates it:
trivy fs --exit-code 1 \
         --severity MEDIUM,HIGH,CRITICAL \
         --format table \
         --output /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt \
         /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
```

**Trivy Parameters Explained**:

* `fs`: Scan filesystem (source code and dependencies)  
* `--exit-code 1`: Fail the build if vulnerabilities found  
* `--severity MEDIUM,HIGH,CRITICAL`: Only report these severity levels  
* `--format table`: Output in table format  
* `--output`: Save report to file for email attachment

**Important**: This step will **FAIL** the build if any vulnerabilities are detected. This is intentional \- it prevents vulnerable code from reaching UAT.

**Application Build**:

bash

```shell
   yarn run build
```

* Compiles Next.js application  
* Creates optimized production build in `.next` directory  
* Only runs if Trivy scan passes

### **Build Process on Jenkins Server(Overview)**

```
Step 1: Clone/Sync prodConf
Location: /home/ubuntu/prodConf
↓
Step 2: Fetch UAT Server IP via AWS CLI
↓
Step 3: Clone/Sync Application Code
Location: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
↓
Step 4: Copy Configuration File
From: /home/ubuntu/prodConf/biddEasy/uat/bidd2.0/next.config.ts
To: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/next.config.ts
↓
Step 5: Install Dependencies
Working Directory: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
Command: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn install
Creates: node_modules/ directory
↓
Step 6: Security Scan
Working Directory: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
Command: trivy fs --exit-code 1 --severity MEDIUM,HIGH,CRITICAL .
Output: /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt
↓
Step 7: Build Application
Working Directory: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application
Command: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn run build
Creates: .next/ directory with compiled code
```

---

## **🖥️ SERVER 2: UAT APPLICATION SERVER (AWS EC2)**

### **Server Details**

```shell
Server Type: AWS EC2 Instance
AWS Region: ap-south-1 (Mumbai)
Instance Tag: Name=bidd-fe-uat
IP Type: Private IP (Dynamic - fetched via AWS CLI)
Operating System: Ubuntu
```

### **How Server IP Is Discovered**

```shell
# Pipeline fetches IP dynamically in Stage 2
aws ec2 describe-instances \
    --region ap-south-1 \
    --filters 'Name=tag:Name,Values=bidd-fe-uat' \
              'Name=instance-state-name,Values=running' \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text

# Example output: 10.0.1.50
# Stored in variable: ${APP_SERVER_IP}
```

### **User Account on UAT Server**

```shell
Username: biddfeuat
Home Directory: /home/biddfeuat
Shell: /bin/bash

# SSH Access
From: Jenkins Build Server (ubuntu user)
To: biddfeuat@<UAT_SERVER_IP>
Authentication: SSH key-based (passwordless)
```

### **Prerequisites Required on UAT Server**

#### **1\. Node.js (Different Version than Build Server)**

```shell
# Node.js version: v20.19.6 (newer than build server)
# Installation path: /home/biddfeuat/.nvm/versions/node/v20.19.6/

Node Binary: /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node
Yarn Binary: /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/yarn
PM2 Binary: /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2
```

**To verify** (SSH to UAT server):

```shell
ssh biddfeuat@<UAT_IP>
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node --version
# Should output: v20.19.6
```

#### **2\. PM2 (Process Manager)**

```shell
# Required for: Running Next.js app as a service
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 --version
# Should output: 5.x.x

# PM2 configuration directory
/home/biddfeuat/.pm2/
```

#### **3\. SSH Server**

```shell
# Required for: Receiving deployments from Jenkins
sudo systemctl status sshd
# Should show: active (running)

# Authorized keys location
/home/biddfeuat/.ssh/authorized_keys
# Must contain public key from Jenkins server (ubuntu@jenkins)
```

### **Directory Structure on UAT Server**

```
/home/biddfeuat/
├── .nvm/
│   └── versions/
│       └── node/
│           └── v20.19.6/
│               └── bin/
│                   ├── node          ← Node.js binary
│                   ├── yarn          ← Yarn binary
│                   └── pm2           ← PM2 process manager
│
├── .pm2/
│   ├── logs/
│   │   ├── uat-bidd-app-out.log     ← Application stdout logs
│   │   └── uat-bidd-app-error.log   ← Application error logs
│   └── dump.pm2                      ← PM2 saved process list
│
└── web.bidd.application/             ← DEPLOY_PATH (Application directory)
    ├── .next/                        ← Copied from Jenkins (compiled code)
    ├── public/                       ← Copied from Jenkins (static files)
    ├── node_modules/                 ← Copied from Jenkins (dependencies)
    ├── package.json                  ← Copied from Jenkins
    ├── yarn.lock                     ← Copied from Jenkins
    └── next.config.ts                ← Copied from Jenkins
```

### **Deployment Process to UAT Server**

**Create Remote Directory**:

bash

```shell
   ssh biddfeuat@${APP_SERVER_IP} "mkdir -p ${DEPLOY_PATH}"
```

#### **Files Copied from Jenkins to UAT Server**

**Transfer Build Artifacts**:

bash

```shell
   scp -r .next public package.json yarn.lock next.config.ts node_modules \
       biddfeuat@${APP_SERVER_IP}:${DEPLOY_PATH}/
```

```shell
Source Location (Jenkins): /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/
Destination Location (UAT): /home/biddfeuat/web.bidd.application/

Files Transferred via SCP:
├── .next/              (entire directory - compiled Next.js build)
├── public/             (entire directory - static assets)
├── node_modules/       (entire directory - all dependencies)
├── package.json        (single file)
├── yarn.lock           (single file)
└── next.config.ts      (single file)

# SCP Command Used:
scp -r -o StrictHostKeyChecking=no \
    .next public package.json yarn.lock next.config.ts node_modules \
    biddfeuat@${APP_SERVER_IP}:/home/biddfeuat/web.bidd.application/
```

#### **Why These Specific Files?**

| File/Directory | Purpose | Size |
| ----- | ----- | ----- |
| `.next/` | Compiled, optimized Next.js code ready to run | \~50-200 MB |
| `public/` | Images, fonts, static files served directly | \~10-50 MB |
| `node_modules/` | All npm packages needed at runtime | \~200-500 MB |
| `package.json` | Defines app metadata and dependencies | \~5 KB |
| `yarn.lock` | Ensures exact dependency versions | \~200 KB |
| `next.config.ts` | Next.js configuration (ports, env vars, etc.) | \~5 KB |

**Note**: Source code (`src/`) is NOT copied because `.next/` contains the compiled version.

### **PM2 Process Management on UAT Server**

#### **PM2 Application Name**

```shell
Application Name: uat.bidd.app
Process Manager: PM2
Start Command: yarn start (runs Next.js in production mode)
```

#### **PM2 Commands Used in Pipeline**

```shell
# Command 1: Try to reload existing app (zero-downtime)
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload uat.bidd.app

# If above fails (app not running), start fresh
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 start \
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/yarn \
--name "uat.bidd.app" -- start

# Command 3: Save PM2 configuration
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
```

#### **Manual PM2 Operations (For Troubleshooting)**

```shell
# SSH to UAT server
ssh biddfeuat@<UAT_IP>

# Check if app is running
pm2 list
# Should show: uat.bidd.app | online | ...

# View logs
pm2 logs uat.bidd.app

# Restart app
pm2 restart uat.bidd.app

# Stop app
pm2 stop uat.bidd.app

# Delete app from PM2
pm2 delete uat.bidd.app

# View detailed info
pm2 show uat.bidd.app

# Monitor CPU/Memory
pm2 monit
```

---

## **📧 SERVER 3: MAIL SERVER(Jenkins Agent)**

### **Mail Configuration**

```shell
Mail Server: Uses Jenkins Build Server's sendmail
From Address: Jenkins <connect@biddeasy.com>
To Address: tech@incredmoney.com
SMTP Configuration: Configured on Jenkins server

# Mail is sent using:
sendmail -t
```

### **When Emails Are Sent**

```
Trigger: Build failure in Stage 3 (Trivy finds vulnerabilities)
Condition: ${WORKSPACE}/trivy_report.txt exists AND contains vulnerabilities
Subject: [SECURITY ALERT] Vulnerabilities Found - ${JOB_NAME} #${BUILD_NUMBER}
Attachment: trivy_report.txt
```

### **Email Content**

```html
From: Jenkins <connect@biddeasy.com>
To: tech@incredmoney.com
Subject: [SECURITY ALERT] Vulnerabilities Found - bidd_ui_uat #123

HTML Body:
- Job Name: bidd_ui_uat
- Build Number: #123
- Branch: uat
- Environment: UAT
- Message: "Deployment blocked due to security findings"

Attachment: trivy_report.txt (contains vulnerability details)
```

---

## **🔄 COMPLETE DATA FLOW**

### **Visual Flow Diagram**

```
┌─────────────────────────────────────────────────────────────────┐
│ JENKINS BUILD SERVER (ubuntu user)                              │
│ /home/ubuntu/                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ STAGE 1: Setup Configuration                                    │
│ ├─ Clone/Sync prodConf repo                                     │
│ └─ Location: /home/ubuntu/prodConf/                             │
│                                                                 │
│ STAGE 2: Fetch UAT Server IP                                    │
│ ├─ AWS CLI query: ec2 describe-instances                        │
│ └─ Result: APP_SERVER_IP = 10.0.1.50                            │
│                                                                 │
│ STAGE 3: Build & Security Scan                                  │
│ ├─ Clone/Sync app repo                                          │
│ │  Location: /home/ubuntu/workspace/bidd_ui_uat/                │
│ │             web.bidd.application/                             │
│ ├─ Copy config file                                             │
│ │  From: /home/ubuntu/prodConf/biddEasy/uat/bidd2.0/            │
│ │        next.config.ts                                         │
│ │  To: /home/ubuntu/workspace/bidd_ui_uat/                      │
│ │      web.bidd.application/next.config.ts                      │
│ ├─ Install dependencies                                         │
│ │  Command: yarn install                                        │
│ │  Creates: node_modules/ (500 MB)                              │
│ ├─ Security scan                                                │
│ │  Command: trivy fs .                                          │
│ │  Output: /home/ubuntu/workspace/bidd_ui_uat/                  │
│ │          trivy_report.txt                                     │
│ │  Action: FAIL BUILD if vulnerabilities found                  │
│ └─ Build application                                            │
│    Command: yarn run build                                      │
│    Creates: .next/ directory (200 MB)                           │
│                                                                 │
└────────────────────┬────────────────────────────────────────────┘
                     │ SCP Transfer
                     │ Files: .next/, public/, node_modules/,
                     │        package.json, yarn.lock, next.config.ts
                     │
                     │ From: ubuntu@jenkins:/home/ubuntu/workspace/
                     │       bidd_ui_uat/web.bidd.application/
                     │ To: biddfeuat@10.0.1.50:/home/biddfeuat/
                     │     web.bidd.application/
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT APPLICATION SERVER (biddfeuat user)                         │
│ AWS EC2: 10.0.1.50 (Private IP)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ STAGE 4: Deploy & Restart                                       │
│ ├─ Receive files via SCP                                        │
│ │  Location: /home/biddfeuat/web.bidd.application/              │
│ └─ Restart PM2 process                                          │
│    Process Name: uat.bidd.app                                   │
│    Command: pm2 reload uat.bidd.app                             │
│    OR: pm2 start yarn --name "uat.bidd.app" -- start            │
│                                                                 │
│ PM2 Logs:                                                       │
│ └─ /home/biddfeuat/.pm2/logs/                                   │
│    ├─ uat-bidd-app-out.log                                      │
│    └─ uat-bidd-app-error.log                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MAIL SERVER (if build fails)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ POST-BUILD: Failure Handler                                     │
│ ├─ Check: trivy_report.txt exists?                              │
│ ├─ Check: Contains vulnerabilities?                             │
│ └─ Send email:                                                  │
│    From: Jenkins <connect@biddeasy.com>                         │
│    To: tech@incredmoney.com                                     │
│    Subject: [SECURITY ALERT] Vulnerabilities Found              │
│    Attachment: trivy_report.txt                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

##  **PATH REFERENCE TABLE**

### **Jenkins Build Server Paths**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| Configuration Repo | `/home/ubuntu/prodConf/` | Environment configs (next.config.ts) |
| Application Source | `/home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/` | Source code, build output |
| Trivy Report | `/home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt` | Security scan results |
| Node.js Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node` | Node runtime |
| Yarn Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn` | Package manager |
| SSH Private Key | `/home/ubuntu/.ssh/id_rsa` | For GitHub and UAT access |

### **UAT Application Server Paths**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| Application Directory | `/home/biddfeuat/web.bidd.application/` | Deployed app (runtime files) |
| Node.js Binary | `/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node` | Node runtime |
| PM2 Binary | `/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2` | Process manager |
| Yarn Binary | `/home/biddfeuat/.nvm/versions/node/v20.19.6/bin/yarn` | Package manager |
| PM2 Logs | `/home/biddfeuat/.pm2/logs/` | Application logs |
| SSH Authorized Keys | `/home/biddfeuat/.ssh/authorized_keys` | Jenkins public key |

---

## **🔧 Whole Jenkins Script (Overview)**

### **Detailed Stage Execution**

```
┌──────────────────────────────────────────────────────────────┐
│ STAGE 1: Setup Configuration (Jenkins Server)               │
└──────────────────────────────────────────────────────────────┘
Server: Jenkins Build Server
User: ubuntu
Working Directory: /home/ubuntu/

Step 1.1: Check if prodConf exists
├─ Path: /home/ubuntu/prodConf
└─ If not exists → git clone git@github.com:incredmoney/prodConf.git

Step 1.2: Sync prodConf repo
├─ cd /home/ubuntu/prodConf
├─ git stash (save local changes)
├─ git fetch origin master
├─ git rebase origin/master
└─ git stash pop || true (restore local changes)

Result: Latest config files available at /home/ubuntu/prodConf/

┌──────────────────────────────────────────────────────────────┐
│ STAGE 2: Initialize & Fetch App IP (Jenkins Server)         │
└──────────────────────────────────────────────────────────────┘
Server: Jenkins Build Server
User: ubuntu

Step 2.1: Query AWS for UAT server IP
├─ Command: aws ec2 describe-instances --region ap-south-1 \
│           --filters 'Name=tag:Name,Values=bidd-fe-uat' \
│                     'Name=instance-state-name,Values=running'
└─ Output: 10.0.1.50 (example)

Step 2.2: Validate IP
├─ Check: IP is not empty
├─ Check: IP is not "None"
└─ If invalid → FAIL PIPELINE

Result: APP_SERVER_IP = 10.0.1.50

┌──────────────────────────────────────────────────────────────┐
│ STAGE 3: Build & Security Scan (Jenkins Server)             │
└──────────────────────────────────────────────────────────────┘
Server: Jenkins Build Server
User: ubuntu
Working Directory: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/

Step 3.1: Clone/Sync application repo
├─ Path: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/
├─ If not exists → git clone -b uat git@github.com:incredmoney/web.bidd.application.git
└─ If exists → git stash, fetch, rebase, stash pop

Step 3.2: Copy environment config
├─ Source: /home/ubuntu/prodConf/biddEasy/uat/bidd2.0/next.config.ts
├─ Destination: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/next.config.ts
└─ Command: cp (overwrites existing)

Step 3.3: Install dependencies
├─ Command: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn install
├─ Reads: package.json, yarn.lock
├─ Creates: node_modules/ directory (~500 MB)
└─ Duration: 2-5 minutes

Step 3.4: Security scan with Trivy
├─ Command: trivy fs --exit-code 1 --severity MEDIUM,HIGH,CRITICAL \
│           --format table --output /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt .
├─ Scans: Source code + node_modules/
├─ Output: /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt
├─ Action: If vulnerabilities found → EXIT CODE 1 → BUILD FAILS
└─ Duration: 1-3 minutes

Step 3.5: Build application (only if scan passes)
├─ Command: /home/ubuntu/.nvm/versions/node/v20.11.1/bin/yarn run build
├─ Creates: .next/ directory (~200 MB)
├─ Compiles: TypeScript → JavaScript, optimizes assets
└─ Duration: 3-7 minutes

Result: Build artifacts ready at /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/

┌──────────────────────────────────────────────────────────────┐
│ STAGE 4: Deploy (Jenkins → UAT Server)                       │
└──────────────────────────────────────────────────────────────┘
Server: Both Jenkins and UAT
User Jenkins: ubuntu
User UAT: biddfeuat

Step 4.1: Create deployment directory (UAT)
├─ Command: ssh biddfeuat@10.0.1.50 "mkdir -p /home/biddfeuat/web.bidd.application"
└─ Creates: /home/biddfeuat/web.bidd.application/ (if not exists)

Step 4.2: Transfer artifacts (Jenkins → UAT)
├─ Command: scp -r -o StrictHostKeyChecking=no \
│           .next public package.json yarn.lock next.config.ts node_modules \
│           biddfeuat@10.0.1.50:/home/biddfeuat/web.bidd.application/
├─ Source: /home/ubuntu/workspace/bidd_ui_uat/web.bidd.application/
├─ Destination: /home/biddfeuat/web.bidd.application/
├─ Total Size: ~700 MB - 1 GB
└─ Duration: 1-3 minutes (depends on network)

Step 4.3: Restart application (UAT Server)
├─ Command: ssh biddfeuat@10.0.1.50
├─ Working Dir: /home/biddfeuat/web.bidd.application/
├─ Step 4.3a: Try reload
│   └─ /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
│      /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload uat.bidd.app
├─ Step 4.3b: If reload fails, start fresh
│   └─ /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
│      /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 start \
│      /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/yarn \
│      --name "uat.bidd.app" -- start
├─ Step 4.3c: Save PM2 configuration
│   └─ /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/node \
│      /home/biddfeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
└─ Duration: 10-30 seconds

Result: Application running at UAT server, accessible via browser

┌──────────────────────────────────────────────────────────────┐
│ POST-BUILD: Cleanup & Alerts (Jenkins Server)                │
└──────────────────────────────────────────────────────────────┘
Server: Jenkins Build Server
User: ubuntu

If Build FAILS:
├─ Check: /home/ubuntu/workspace/bidd_ui_uat/trivy_report.txt exists?
├─ Check: Report contains vulnerabilities?
├─ If yes:
│   ├─ Read report content
│   ├─ Compose HTML email
│   ├─ Attach trivy_report.txt
│   └─ Send via: sendmail -t
│       From: Jenkins <connect@biddeasy.com>
│       To: tech@incredmoney.com
│       Subject: [SECURITY ALERT] Vulnerabilities Found
└─ Remove report
```

# **BIDD API UAT Deployment Pipeline \- Complete Documentation**

## **📋 Quick Overview**

This pipeline deploys the **BIDD API (Backend)** to UAT environment with flexible deployment options.

**Pipeline involves 3 components:**

1. **Jenkins Build Server** \- Where code is synced and scanned  
2. **AWS EC2 (UAT Servers)** \- Multiple backend servers where API runs  
3. **Mail Server** \- Sends security alerts (uses Jenkins server's sendmail)

### **Pipeline Flow**

```
Git Repositories (api + Communications + Config)
↓
Jenkins Build Server (Sync & Scan)
↓
Trivy Security Scan
↓
Deploy to Multiple UAT API Servers (Sequential)
↓
PM2 Reload on Each Server
```

**SERVER 1: JENKINS BUILD SERVER**

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

* `git@github.com:incredmoney/IncredMoney_api.git`  
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
        ├── IncredMoney_api/      ← Main API repository
        │   ├── conf/
        │   │   ├── conf.json         ← API configuration
        │   │  
        │   ├── app.bidd.communications/  ← Git submodule
        │   │   └── (communication module code)
        │   ├── node_modules/         ← After npm install
        │   ├── package.json
        │   └── (API source code)
        │
        └── api_trivy_report.txt  ← Trivy scan output
```

---

## **📦 Where Repositories Are Cloned**

### **Repository 1: API (Main Repository)**

```shell
Repository URL: git@github.com:incredmoney/IncredMoney_api.git
Clone Location: ${WORKSPACE}/IncredMoney_api
Full Path: /home/ubuntu/workspace/<JOB_NAME>/IncredMoney_api
Branch: develop
Cloned By: Stage 2 (Code Sync + NPM + Trivy)

# Clone command:
git clone -b develop git@github.com:incredmoney/IncredMoney_api.git \
    ${WORKSPACE}/IncredMoney_api
```

### **Repository 2: Communications Module (Git Submodule)**

```shell
Repository URL: git@github.com:incredmoney/app.bidd.communications.git
Clone Location: ${WORKSPACE}/IncredMoney_api/app.bidd.communications
Full Path: /home/ubuntu/workspace/<JOB_NAME>/IncredMoney_api/app.bidd.communications
Branch: develop
Type: Git Submodule
Cloned By: Stage 2 (as part of api repo submodule init)

# Submodule command:
cd IncredMoney_api
git submodule add -b develop \
    git@github.com:incredmoney/app.bidd.communications.git \
    app.bidd.communications
```

### **Repository 3: Configuration Repository (prodConf)**

```shell
Repository URL: git@github.com:incredmoney/prodConf.git
Clone Location: Cloned on UAT servers (not on Jenkins)
Branch: master
Cloned By: Stage 4 (Deploy to App Servers) - only if WITH_CONF or WITH_COMMS is true

# This repo is cloned on each UAT server:
Location on UAT: /home/biddbeuat/prodConf/

Contains:
- /home/biddbeuat/prodConf/biddEasy/uat/api/conf.json

```

---

## **🎯 Pipeline Parameters (Deployment Modes)**

The pipeline has **3 boolean parameters** that control what gets deployed:

| Parameter | Default | Description | What It Does |
| ----- | ----- | ----- | ----- |
| `WITH_CONF` | `false` | Deploy API configuration (conf.json) | Copies `conf.json` from prodConf to API |
|  |  |  |  |
| `WITH_COMMS` | `false` | Deploy API communications module | Updates communications submodule |

### **Deployment Scenarios**

#### **Scenario 1: Code-Only Deployment (All parameters \= false)**

```
✅ API code updated
❌ Configuration files NOT updated
❌ Communications module NOT updated
```

#### **Scenario 2: Code \+ Configuration (WITH\_CONF \= true,WITH\_COMMS=false)**

```
✅ API application code is updated	
✅ conf.json updated from prodConf
❌ Communications module NOT updated
```

#### **Scenario 3: Full Deployment (WITH\_COMMS= true,WITH\_CONF=true)**

```
✅ API code updated
✅ conf.json updated
✅ Communications module updated
```

---

## **🔄 PIPELINE STAGES**

### **Stage 1: Validate Parameters**

```
stage('Validate Parameters') {
    steps {
        echo "Deployment Mode: WITH_CONF=${params.WITH_CONF}, WITH_COMMS=${params.WITH_COMMS}"
    }
}
```

**Purpose:** Display which deployment mode is selected

**Output Example:**

```
Deployment Mode: WITH_CONF=true, WITH_COMMS=true
```

---

### **Stage 2: Code Sync \+ NPM \+ Trivy**

**Server:** Jenkins Build Server  
 **User:** ubuntu  
 **Working Directory:** `${WORKSPACE}/IncredMoney_api`

#### **Step 2.1: Clone/Sync API Repository**

```shell
# If directory doesn't exist, clone it
if [ ! -d "${WORKSPACE}/IncredMoney_api" ]; then
    git clone -b develop git@github.com:incredmoney/IncredMoney_api.git \
        ${WORKSPACE}/IncredMoney_api
fi

# If exists, sync it
cd ${WORKSPACE}/IncredMoney_api
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

**Why this step?**

* Removes old submodule `app.icm.communications` (if exists)  
* Adds new submodule `app.bidd.communications`  
* Uses develop branch

#### **Step 2.3: Install Dependencies**

```shell
npm install
```

**Creates:** `node_modules/` directory in `${WORKSPACE}/IncredMoney_api/`

#### **Step 2.4: Trivy Security Scan**

```shell
trivy fs . \
    --severity LOW,MEDIUM,HIGH,CRITICAL \
    --exit-code 0 \
    --format table \
    --output ${WORKSPACE}/api_trivy_report.txt
```

**Trivy Configuration:**

| Parameter | Value | Meaning |
| ----- | ----- | ----- |
| `fs .` | Current directory | Scan entire api repo |
| `--severity` | LOW,MEDIUM,HIGH,CRITICAL | All severity levels |
| `--exit-code 0` | **Does NOT fail build** | Only logs vulnerabilities |
| `--format table` | Table format | Human-readable output |
| `--output` | `${WORKSPACE}/api_trivy_report.txt` | Save to file |

**⚠️ Important Difference from UI Pipeline:**

* UI Pipeline: `--exit-code 1` (FAILS build if vulnerabilities found)  
* api Pipeline: `--exit-code 0` (DOES NOT fail build, only sends email alert)

### **Where Trivy Report Is Generated**

```shell
Report Location: ${WORKSPACE}/api_trivy_report.txt
Full Path: /home/ubuntu/workspace/<JOB_NAME>/api_trivy_report.txt

Generated By: Stage 2 (Code Sync + NPM + Trivy)
Used By: Post-build failure api (email attachment)
Exit Code: 0 (does not fail pipeline)
```

---

### **Stage 3: Fetch API Server IPs**

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

**Server:** UAT API Servers (Multiple)  
 **User:** biddbeuat  
 **Method:** SSH from Jenkins → Git pull on each server

**Deployment Strategy:**

```
For each IP in APP_SERVER_IPS:
    SSH to server
    Git pull API code
    Update configuration (if WITH_CONF enabled)
    Update communications (if WITH_COMMS enabled)
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

#### **Step 4.1: Sync API Code on UAT Server**

```shell
cd /home/biddbeuat/IncredMoney_api
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
    echo "Updating Configuration..."
    
    # Clone/sync prodConf repo on UAT server
    cd /home/biddbeuat
    [ ! -d prodConf ] && git clone -b master git@github.com:incredmoney/prodConf.git
    cd prodConf
    git stash
    git fetch origin master
    git rebase origin/master
    git stash pop || true
    
    # Copy configuration file
    cp /home/biddbeuat/prodConf/biddEasy/uat/api/conf.json \
       /home/biddbeuat/IncredMoney_api/conf/conf.json
fi

```

**File Copy:**

```
Source: /home/biddbeuat/prodConf/biddEasy/uat/api/conf.json
Destination: /home/biddbeuat/IncredMoney_api/conf/conf.json
```

#### 

#### **Step 4.4: Update Communications Module (If WITH\_COMMS \= true)**

```shell
if [ "${params.WITH_COMMS}" = "true" ]; then
    echo "Updating Communications Module..."
    
    cd /home/biddbeuat/IncredMoney_api/app.bidd.communications
    git stash
    git fetch origin develop
    git rebase origin/develop
    git stash pop || true
fi

```

**Submodule Location:**

```
/home/biddbeuat/IncredMoney_api/app.bidd.communications/
```

#### **Step 4.5: Reload PM2 Process**

```shell
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload api
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 save

```

**PM2 App Name:** `api`

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

* `API repo: git@github.com:incredmoney/IncredMoney_api.git`  
* `Communications repo: git@github.com:incredmoney/app.bidd.communications.git`  
* `Config repo: git@github.com:incredmoney/prodConf.git`  
* 

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
│   │   ├── api-out.log              ← Application stdout logs
│   │   └── api-error.log            ← Application error logs
│   └── dump.pm2                     ← PM2 saved process list
│
├── prodConf/                        ← Config repository (cloned if WITH_CONF = true)
│   └── biddEasy/
│       └── uat/
│           └── api/
│               └── conf.json        ← API configuration
│
└── IncredMoney_api/                 ← Main API application
    ├── conf/
    │   └── conf.json                ← Runtime configuration (copied from prodConf)
    ├── app.bidd.communications/     ← Git submodule
    │   └── (communication module code)
    ├── node_modules/                ← NPM dependencies
    ├── package.json
    ├── index.js                     ← Main entry point
    └── (other API source files)

```

---

## **🔄 PM2 Process Management on UAT Servers**

### **PM2 Application Configuration**

```shell
Application Name: api
Process Manager: PM2
Working Directory: /home/biddbeuat/IncredMoney_api
Entry Point: index.js (or as defined in package.json)

```

### **PM2 Commands Used in Pipeline**

```shell
# Reload application (zero-downtime restart)
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload api

# Save PM2 configuration
/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
```

### **Manual PM2 Operations**

```shell
# SSH to any UAT server
ssh biddbeuat@<UAT_IP>

# Check if app is running
pm2 list
# Should show: api | online | ...

# View logs
pm2 logs api

# View last 100 lines
pm2 logs api --lines 100

# Restart app
pm2 restart api

# Stop app
pm2 stop api

# Delete app
pm2 delete api

# Start app manually
pm2 start index.js --name api

# Monitor CPU/Memory
pm2 monit

# View detailed info
pm2 show api

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
Condition: api_trivy_report.txt exists AND contains vulnerabilities
Subject: [SECURITY ALERT] Vulnerabilities Found - ${JOB_NAME} #${BUILD_NUMBER}
Attachment: api_trivy_report.txt

```

### **Email Alert Logic**

```
post {
    failure {
        script {
            // Check if Trivy report exists and has vulnerabilities
            def hasApiVuls = checkVuls("${WORKSPACE}/api_trivy_report.txt")
            
            if (hasApiVuls) {
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
grep -E 'Total: [1-9]|[0-9]+ (CRITICAL|HIGH|MEDIUM|LOW)' api_trivy_report.txt
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
- Findings: API
- Message: "Jenkins blocked deployment"

Attachment: api_trivy_report.txt
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
│ └─ Display: WITH_CONF, WITH_COMMS                              │
│                                                                  │
│ STAGE 2: Code Sync + NPM + Trivy                               │
│ ├─ Clone/Sync API repo                                          │
│ │  Location: ${WORKSPACE}/IncredMoney_api/                     │
│ ├─ Initialize Communications submodule                          │
│ │  Location: ${WORKSPACE}/IncredMoney_api/                     │
│ │             app.bidd.communications/                          │
│ ├─ NPM install                                                  │
│ │  Creates: node_modules/                                       │
│ └─ Trivy security scan                                          │
│    Output: ${WORKSPACE}/api_trivy_report.txt                   │
│    Exit Code: 0 (does not fail build)                          │
│                                                                  │
│ STAGE 3: Fetch API Server IPs                                  │
│ └─ AWS CLI: Get IPs of bidd-be-uat instances                   │
│    Result: APP_SERVER_IPS = "10.0.2.10 10.0.2.11 10.0.2.12"    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Sequential SSH Deployment
                     │ (No file copy - Git pull on each server)
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT API SERVER 1: 10.0.2.10 (biddbeuat user)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ STAGE 4: Deploy to App Servers                                 │
│ ├─ Git pull API code                                            │
│ │  Location: /home/biddbeuat/IncredMoney_api/                 │
│ │  Branch: develop                                              │
│ ├─ NPM install                                                  │
│ │                                                               │
│ ├─ [IF WITH_CONF = true]                                       │
│ │  Clone/sync: /home/biddbeuat/prodConf/                       │
│ │  Copy: prodConf/.../conf.json → API/conf/conf.json          │
│ │                                                               │
│ ├─ [IF WITH_COMMS = true]                                      │
│ │  Git pull communications submodule                            │
│ │  Location: /home/biddbeuat/IncredMoney_api/                 │
│ │             app.bidd.communications/                          │
│ │                                                               │
│ └─ PM2 reload api                                               │
│    PM2 save                                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Deploy to next server
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT API SERVER 2: 10.0.2.11 (biddbeuat user)                   │
│ (Same steps as Server 1)                                        │
└─────────────────────────────────────────────────────────────────┘
                     │
                     │ Deploy to next server
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ UAT API SERVER 3: 10.0.2.12 (biddbeuat user)                   │
│ (Same steps as Server 1)                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MAIL SERVER (if build fails)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ POST-BUILD: Failure api                                     │
│ ├─ Check: api_trivy_report.txt exists?                          │
│ ├─ Check: Contains vulnerabilities?                             │
│ └─ If yes, send email:                                          │
│    From: Jenkins <connect@biddeasy.com>                         │
│    To: rajat.sah@incredmoney.com                                │
│    Subject: [SECURITY ALERT] Vulnerabilities Found              │
│    Attachment: api_trivy_report.txt                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

```

---

## **📊 PATH REFERENCE TABLE**

### **Jenkins Build Server Paths**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| API Repository | `${WORKSPACE}/IncredMoney_api/` | API source code |
| Communications Submodule | `${WORKSPACE}/IncredMoney_api/app.bidd.communications/` | Communications module |
| Trivy Report | `${WORKSPACE}/api_trivy_report.txt` | Security scan results |
| Node.js Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/node` | Node runtime |
| NPM Binary | `/home/ubuntu/.nvm/versions/node/v20.11.1/bin/npm` | Package manager |
| SSH Private Key | `/home/ubuntu/.ssh/id_rsa` | For GitHub and UAT access |

### **UAT API Server Paths (Each Server)**

| Purpose | Path | Contents |
| ----- | ----- | ----- |
| API Application | `/home/biddbeuat/IncredMoney_api/` | Running API application |
| Communications Submodule | `/home/biddbeuat/IncredMoney_api/app.bidd.communications/` | Communications module |
| Configuration Repository | `/home/biddbeuat/prodConf/` | Environment configs |
| conf.json Source | `/home/biddbeuat/prodConf/biddEasy/uat/api/conf.json` | API config source |
| conf.json Destination | `/home/biddbeuat/IncredMoney_api/conf/conf.json` | API config (runtime) |
| Node.js Binary | `/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/node` | Node runtime |
| PM2 Binary | `/home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2` | Process manager |
| PM2 Logs | `/home/biddbeuat/.pm2/logs/` | Application logs (`api-out.log`, `api-error.log`) |
| SSH Authorized Keys | `/home/biddbeuat/.ssh/authorized_keys` | Jenkins public key |
| **Purpose** | **Path** | **Contents** |
| API Application | `/home/biddbeuat/IncredMoney_api/` | Running API application |

---

**DETAILED STAGE EXECUTION**

### **STAGE 1: Validate Parameters (Jenkins Server)**

```
Server: Jenkins Build Server
User: ubuntu

Step 1.1: Display deployment mode
└─ Echo: WITH_CONF=${params.WITH_CONF}
         WITH_COMMS=${params.WITH_COMMS}

Result: User knows which components will be deployed
```

---

### **STAGE 2: Code Sync \+ NPM \+ Trivy (Jenkins Server)**

```
Server: Jenkins Build Server
User: ubuntu
Working Directory: ${WORKSPACE}/IncredMoney_api/

Step 2.1: Clone/Sync Api Repository
├─ Check if exists: ${WORKSPACE}/IncredMoney_api
├─ If not exists:
│   └─ git clone -b develop git@github.com:incredmoney/IncredMoney_api.git
├─ If exists:
│   ├─ cd ${WORKSPACE}/IncredMoney_api
│   ├─ git stash
│   ├─ git fetch origin develop
│   ├─ git rebase origin/develop
│   └─ git stash pop || true

Step 2.2: Initialize Communications Submodule
├─ Check if submodule exists: app.bidd.communications/
├─ If not exists:
│   ├─ Remove old submodule: git rm -rf app.icm.communications || true
│   └─ Add new submodule:
│       git submodule add -b develop \
│           git@github.com:incredmoney/app.bidd.communications.git \
│           app.bidd.communications

Step 2.3: Install NPM Dependencies
├─ Command: npm install
├─ Creates: node_modules/ directory
└─ Duration: 1-3 minutes

Step 2.4: Trivy Security Scan
├─ Command: trivy fs . --severity LOW,MEDIUM,HIGH,CRITICAL \
│           --exit-code 0 --format table \
│           --output ${WORKSPACE}/api_trivy_report.txt
├─ Scans: All files in API repo + node_modules/
├─ Output: ${WORKSPACE}/api_trivy_report.txt
├─ Exit Code: 0 (does NOT fail build even if vulnerabilities found)
└─ Duration: 1-3 minutes

Result: 
- API code synced
- Communications submodule initialized
- Dependencies installed
- Security scan completed (report saved for post-build email)
```

---

### **STAGE 3: Fetch API Server IPs (Jenkins Server)**

```
Server: Jenkins Build Server
User: ubuntu

Step 3.1: Query AWS for running UAT API servers
├─ Command: aws ec2 describe-instances \
│           --filters "Name=tag:Name,Values=bidd-be-uat" \
│                     "Name=instance-state-name,Values=running" \
│           --query "Reservations[].Instances[].PrivateIpAddress" \
│           --output text
├─ Output Example: "10.0.2.10 10.0.2.11 10.0.2.12"
└─ Stored in: ${APP_SERVER_IPS}

Step 3.2: Validate IPs
├─ Check: At least one IP returned
└─ If no IPs: error "No running API servers found"

Result: APP_SERVER_IPS = "10.0.2.10 10.0.2.11 10.0.2.12"
```

---

### **STAGE 4: Deploy to App Servers \- Sequential (UAT Servers)**

```
Server: Multiple UAT API Servers (one at a time)
User: biddbeuat
Method: SSH from Jenkins → Execute commands on each server

For EACH IP in APP_SERVER_IPS:
    
    ┌─────────────────────────────────────────────────────────┐
    │ Deploying to Server: ${IP}                             │
    └─────────────────────────────────────────────────────────┘
    
    Step 4.1: SSH to Server
    ├─ Command: ssh -o StrictHostKeyChecking=no biddbeuat@${IP}
    └─ All following steps execute ON THE UAT SERVER
    
    Step 4.2: Sync api Code
    ├─ cd /home/biddbeuat/IncredMoney_api
    ├─ git stash
    ├─ git fetch origin develop
    ├─ git rebase origin/develop
    ├─ git stash pop || true
    └─ npm install
    
    Step 4.3: Update conf.json [IF WITH_CONF = true]
    ├─ cd /home/biddbeuat
    ├─ Check if prodConf exists
    │   └─ If not: git clone -b master git@github.com:incredmoney/prodConf.git
    ├─ cd prodConf
    ├─ git stash
    ├─ git fetch origin master
    ├─ git rebase origin/master
    ├─ git stash pop || true
    └─ Copy file:
        cp /home/biddbeuat/prodConf/biddEasy/uat/api/conf.json \
           /home/biddbeuat/IncredMoney_api/conf/conf.json
    
    Step 4.4: Update Communications [IF WITH_COMMS = true]
    ├─ cd /home/biddbeuat/IncredMoney_api/app.bidd.communications
    ├─ git stash
    ├─ git fetch origin develop
    ├─ git rebase origin/develop
    └─ git stash pop || true
    
    Step 4.5: Reload PM2 Application
    ├─ /home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 reload api
    └─ /home/biddbeuat/.nvm/versions/node/v20.19.6/bin/pm2 save
    
    Result: Server ${IP} updated successfully
    
    Move to next IP and repeat...

Final Result: All UAT API servers updated sequentially
```

---

### **POST-BUILD: Failure api (Jenkins Server)**

```
Server: Jenkins Build Server
User: ubuntu

Trigger: Pipeline fails at any stage

Step 1: Check for Security Vulnerabilities
├─ Check if exists: ${WORKSPACE}/api_trivy_report.txt
└─ Check if contains vulnerabilities:
    grep -E 'Total: [1-9]|[0-9]+ (CRITICAL|HIGH|MEDIUM|LOW)' \
        api_trivy_report.txt

Step 2: Send Email Alert (if vulnerabilities found)
├─ Compose HTML email body with:
│   ├─ Job Name
│   ├─ Build Number
│   ├─ Environment: UAT
│   └─ Findings: api
├─ Attach: api_trivy_report.txt
└─ Send via: sendmail -t
    From: Jenkins <connect@biddeasy.com>
    To: rajat.sah@incredmoney.com
    Subject: [SECURITY ALERT] Vulnerabilities Found - ${JOB_NAME} #${BUILD_NUMBER}

Step 3: If No Vulnerabilities
└─ Echo: "FAILURE CAUSE: General Build Failure"
         "No security vulnerabilities detected"

Result: 
- Team notified if security issues found
- Build failure logged for troubleshooting
```

---


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
