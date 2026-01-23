# Jenkins Day 4 — Real-World CI Pipeline (Frontend + Node + FastAPI) | Tech With Diwana

<p align="center">
  <a href="#"><img alt="Jenkins" src="https://img.shields.io/badge/Jenkins-CI%2FCD-blue?style=for-the-badge"></a>
  <a href="#"><img alt="GitHub Webhook" src="https://img.shields.io/badge/GitHub-Webhook-black?style=for-the-badge"></a>
  <a href="#"><img alt="NGINX Reverse Proxy" src="https://img.shields.io/badge/NGINX-Reverse%20Proxy-green?style=for-the-badge"></a>
  <a href="#"><img alt="HTTPS" src="https://img.shields.io/badge/HTTPS-Let's%20Encrypt-success?style=for-the-badge"></a>
  <a href="#"><img alt="Docker Build" src="https://img.shields.io/badge/Docker-Image%20Build-informational?style=for-the-badge"></a>
</p>

---

## Overview (Day 4)

Day 4 is **pure CI**.

We will build a production-style Jenkins Pipeline that:

- pulls code from GitHub (webhook-triggered)
- runs clean build steps for:
  - `frontend` (Nginx static app)
  - `backend-node` (Node.js API gateway)
  - `backend-fastapi` (Python FastAPI vector service)
- runs API checks + UI/E2E test scaffolding
- keeps Jenkins secure:
  - Jenkins **8080 stays private (localhost only)**
  - NGINX serves Jenkins publicly on **443**
  - GitHub webhook hits **/github-webhook/** via HTTPS

> Day 4 = **Build + Test** only  
> Deployment to Kubernetes = **Day 5**

---

## Repo Structure (Target)

We are working inside:

```
kubernetes-zero-to-hero/
└── day13-techwithdiwana_gcp_llm_cluster/
    ├── backend-fastapi/
    ├── backend-node/
    ├── frontend/
    ├── k8s/
    ├── Jenkinsfile
    ├── db/
    │   ├── README.md
    │   └── migrations/
    │       └── 001_init.sql
    └── tests/
        ├── api/
        │   └── smoke.sh
        └── ui/
            ├── package.json
            ├── playwright.config.js
            └── smoke.spec.js
### What each folder contains

- **backend-fastapi/** → FastAPI backend service (API)
- **backend-node/** → Node.js backend service (API)
- **frontend/** → Frontend UI application
- **k8s/** → Kubernetes manifests (Deployments, Services, Ingress, ConfigMaps, etc.)
- **Jenkinsfile** → Jenkins CI pipeline definition (Build + Test for Day4)
- **db/** → Database init/migration scripts
- **tests/api/smoke.sh** → API smoke test script (quick health checks)
- **tests/ui/** → Playwright UI automation tests

> **Note:** Day4 focus is **CI (Build + Test only)**.  
> Kubernetes deployment will be covered in **Day5** using the same repo structure.


```

---

# Day 3 (Already Done) — Keep These Steps AS-IS

> Do not skip Day 3. Day 4 depends on it.

## Goal (Day 3)

- Jenkins VM on GCP
- Jenkins secured behind NGINX + HTTPS
- Port **8080 not public**
- GitHub webhook triggers Jenkins

## PART 0 — PowerShell Variables (Windows)

```powershell
$PROJECT_ID="YOUR_PROJECT_ID"
$ZONE="asia-south1-a"
$VM_NAME="jenkins-vm"
$TAG="jenkins-server"
$MY_IP="122.161.52.244/32"
$MACHINE_TYPE="e2-medium"
$DISK_SIZE="50GB"
$DOMAIN="jenkins.techwithdiwana.com"
```

Set project:

```powershell
gcloud config set project $PROJECT_ID
```

## PART 1 — Create VM (GCP)

```powershell
gcloud compute instances create $VM_NAME `
  --zone $ZONE `
  --machine-type $MACHINE_TYPE `
  --image-family ubuntu-2204-lts `
  --image-project ubuntu-os-cloud `
  --boot-disk-size $DISK_SIZE `
  --tags $TAG
```

Get VM public IP:

```powershell
gcloud compute instances describe $VM_NAME --zone $ZONE --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

## PART 2 — Secure Firewall Rules (NO 8080 Public)

### 2.1 Allow SSH only from your IP

```powershell
gcloud compute firewall-rules create jenkins-allow-ssh-myip `
  --direction=INGRESS `
  --priority=1000 `
  --network=default `
  --action=ALLOW `
  --rules=tcp:22 `
  --source-ranges=$MY_IP `
  --target-tags=$TAG
```

### 2.2 Allow HTTP/HTTPS from Internet (for NGINX + Webhook)

```powershell
gcloud compute firewall-rules create jenkins-allow-http-https `
  --direction=INGRESS `
  --priority=1000 `
  --network=default `
  --action=ALLOW `
  --rules="tcp:80,tcp:443" `
  --source-ranges=0.0.0.0/0 `
  --target-tags=$TAG
```

IMPORTANT:
- Do NOT create firewall rule for `tcp:8080`

## PART 3 — SSH into VM

```bash
gcloud compute ssh $VM_NAME --zone $ZONE
```

## PART 4 — Install Jenkins + NGINX + Git

```bash
sudo apt update -y
sudo apt install -y fontconfig openjdk-17-jre git nginx
```

Install Jenkins:

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update -y
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

Check Jenkins:

```bash
sudo systemctl status jenkins --no-pager
```

## PART 5 — Make Jenkins 8080 LOCALHOST ONLY (Secure)

```bash
sudo mkdir -p /etc/systemd/system/jenkins.service.d
```

```bash
sudo tee /etc/systemd/system/jenkins.service.d/override.conf > /dev/null <<'EOF'
[Service]
Environment="JENKINS_LISTEN_ADDRESS=127.0.0.1"
EOF
```

Reload + restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Verify:

```bash
sudo ss -lntp | grep 8080
```

Expected:

```
127.0.0.1:8080
```

---

# Day 4 — CI Pipeline (Scratch to Production Style)

## PART 1 — Install Day-4 CI Tooling on Jenkins VM

Run on Jenkins VM:

```bash
sudo apt update -y
sudo apt install -y docker.io curl unzip jq python3 python3-pip nodejs npm
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Allow Jenkins user to run Docker:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Verify:

```bash
docker --version
node --version
npm --version
python3 --version
```

## PART 2 — Add GitHub Credentials in Jenkins (Secure)

Minimum PAT permissions:

- Contents: Read-only
- Metadata: Read-only

Add to Jenkins:

- Kind: Username with password
- Username: techwithdiwana
- Password: GitHub PAT
- ID: github-creds

## PART 3 — Create Jenkins Pipeline Job (Day 4)

- Name: day4-ci-pipeline
- Type: Pipeline
- Definition: Pipeline script from SCM
- Repo: https://github.com/techwithdiwana/kubernetes-zero-to-hero.git
- Branch: */main
- Script Path: day13-techwithdiwana_gcp_llm_cluster/Jenkinsfile

## PART 4 — Jenkinsfile (CI Only: Build + Basic Tests)

Create:

`day13-techwithdiwana_gcp_llm_cluster/Jenkinsfile`

```groovy
pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    ROOT_DIR = "day13-techwithdiwana_gcp_llm_cluster"
    FRONTEND_DIR = "${ROOT_DIR}/frontend"
    NODE_DIR = "${ROOT_DIR}/backend-node"
    FASTAPI_DIR = "${ROOT_DIR}/backend-fastapi"

    FRONTEND_IMAGE = "twd-frontend"
    NODE_IMAGE = "twd-backend-node"
    FASTAPI_IMAGE = "twd-backend-fastapi"

    IMAGE_TAG = "ci-${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'ls -la'
      }
    }

    stage('Validate Repo Paths') {
      steps {
        sh '''
          set -e
          test -d "${FRONTEND_DIR}"
          test -d "${NODE_DIR}"
          test -d "${FASTAPI_DIR}"
          echo "Repo paths look OK"
        '''
      }
    }

    stage('Frontend: Docker Build') {
      steps {
        sh '''
          set -e
          cd "${FRONTEND_DIR}"
          docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Backend Node: Docker Build') {
      steps {
        sh '''
          set -e
          cd "${NODE_DIR}"
          docker build -t ${NODE_IMAGE}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Backend FastAPI: Docker Build') {
      steps {
        sh '''
          set -e
          cd "${FASTAPI_DIR}"
          docker build -t ${FASTAPI_IMAGE}:${IMAGE_TAG} .
        '''
      }
    }

    stage('API Smoke Check (Local Container Run)') {
      steps {
        sh '''
          set -e

          docker rm -f twd_fastapi_ci || true
          docker run -d --name twd_fastapi_ci -p 9001:8000 ${FASTAPI_IMAGE}:${IMAGE_TAG}

          sleep 5
          curl -sS http://127.0.0.1:9001/healthz | jq .

          docker rm -f twd_fastapi_ci || true
        '''
      }
    }

    stage('Artifact Summary') {
      steps {
        sh '''
          echo "Built images:"
          docker images | head -n 20
        '''
      }
    }
  }

  post {
    always {
      echo "CI pipeline finished"
    }
  }
}
```

## PART 5 — Pre-Flight Checklist

```bash
docker --version
git --version
node --version
npm --version
python3 --version
pip3 --version
```

Docker running:

```bash
sudo systemctl status docker --no-pager
```

Jenkins docker group:

```bash
groups jenkins
```

Webhook status: GitHub → Webhooks → Recent Deliveries

---

## Completion Criteria (Day 4)

- Webhook triggers Jenkins Pipeline
- Frontend + Node + FastAPI images build successfully
- FastAPI `/healthz` returns OK during CI
- Jenkins 8080 stays private (localhost only)
- Jenkins opens via HTTPS domain
