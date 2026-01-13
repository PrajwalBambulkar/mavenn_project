# **BIDD UAT Pipeline Complete Documentation**

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

### **Where Repositories Are Cloned**

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

aws ec2 describe-instances \\  
\--filters Name=tag:Name,Values=bidd-fe-uat \\  
Name=instance-state-name,Values=running

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
│ /home/ubuntu/                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ STAGE 1: Setup Configuration                                    │
│ ├─ Clone/Sync prodConf repo                                     │
│ └─ Location: /home/ubuntu/prodConf/                             │
│                                                                  │
│ STAGE 2: Fetch UAT Server IP                                    │
│ ├─ AWS CLI query: ec2 describe-instances                        │
│ └─ Result: APP_SERVER_IP = 10.0.1.50                            │
│                                                                  │
│ STAGE 3: Build & Security Scan                                  │
│ ├─ Clone/Sync app repo                                          │
│ │  Location: /home/ubuntu/workspace/bidd_ui_uat/               │
│ │             web.bidd.application/                             │
│ ├─ Copy config file                                             │
│ │  From: /home/ubuntu/prodConf/biddEasy/uat/bidd2.0/           │
│ │        next.config.ts                                         │
│ │  To: /home/ubuntu/workspace/bidd_ui_uat/                     │
│ │      web.bidd.application/next.config.ts                      │
│ ├─ Install dependencies                                         │
│ │  Command: yarn install                                        │
│ │  Creates: node_modules/ (500 MB)                              │
│ ├─ Security scan                                                │
│ │  Command: trivy fs .                                          │
│ │  Output: /home/ubuntu/workspace/bidd_ui_uat/                 │
│ │          trivy_report.txt                                     │
│ │  Action: FAIL BUILD if vulnerabilities found                  │
│ └─ Build application                                            │
│    Command: yarn run build                                      │
│    Creates: .next/ directory (200 MB)                           │
│                                                                  │
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
│                                                                  │
│ STAGE 4: Deploy & Restart                                       │
│ ├─ Receive files via SCP                                        │
│ │  Location: /home/biddfeuat/web.bidd.application/             │
│ └─ Restart PM2 process                                          │
│    Process Name: uat.bidd.app                                   │
│    Command: pm2 reload uat.bidd.app                             │
│    OR: pm2 start yarn --name "uat.bidd.app" -- start            │
│                                                                  │
│ PM2 Logs:                                                        │
│ └─ /home/biddfeuat/.pm2/logs/                                   │
│    ├─ uat-bidd-app-out.log                                      │
│    └─ uat-bidd-app-error.log                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MAIL SERVER (if build fails)                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ POST-BUILD: Failure Handler                                     │
│ ├─ Check: trivy_report.txt exists?                              │
│ ├─ Check: Contains vulnerabilities?                             │
│ └─ Send email:                                                  │
│    From: Jenkins <connect@biddeasy.com>                         │
│    To: tech@incredmoney.com                                     │
│    Subject: [SECURITY ALERT] Vulnerabilities Found              │
│    Attachment: trivy_report.txt                                 │
│                                                                  │
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
│ STAGE 4: Deploy (Jenkins → UAT Server)                      │
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
│ POST-BUILD: Cleanup & Alerts (Jenkins Server)               │
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
