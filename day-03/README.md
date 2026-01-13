# Day 3 — Git Integration & Build Triggers (Jenkins + GitHub Webhook) 🚀

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-blue?style=for-the-badge)
![GitHub](https://img.shields.io/badge/GitHub-Webhooks-black?style=for-the-badge)
![GCP](https://img.shields.io/badge/GCP-Compute%20Engine-orange?style=for-the-badge)
![NGINX](https://img.shields.io/badge/NGINX-Reverse%20Proxy-green?style=for-the-badge)
![HTTPS](https://img.shields.io/badge/HTTPS-Let's%20Encrypt-brightgreen?style=for-the-badge)
![Secure](https://img.shields.io/badge/Security-8080%20Localhost%20Only-red?style=for-the-badge)

> **Goal (Day 3):** Connect Jenkins with GitHub, configure credentials, enable webhook triggers, and secure Jenkins by keeping **port 8080 private** (localhost only) behind **NGINX + HTTPS**.

---

## 🎯 What You Will Achieve (Day 3 Outcomes)

By the end of Day 3, you will have:

- ✅ A Jenkins VM on **GCP Compute Engine**
- ✅ **Firewall secured** (no public access to `8080`)
- ✅ Jenkins running on **localhost:8080 only**
- ✅ NGINX reverse proxy on **80/443**
- ✅ HTTPS enabled via **Certbot / Let's Encrypt**
- ✅ A Jenkins Freestyle Job that pulls code from GitHub
- ✅ GitHub Webhook that triggers Jenkins automatically on every push

---

## 🧩 PART 0 — PowerShell: Variables Set (Windows)

> Run these in **Windows PowerShell** (or Terminal) on your local machine.

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

### Set GCP project

```powershell
gcloud config set project $PROJECT_ID
```

---

## 🖥️ PART 1 — Create VM (GCP)

```powershell
gcloud compute instances create $VM_NAME `
  --zone $ZONE `
  --machine-type $MACHINE_TYPE `
  --image-family ubuntu-2204-lts `
  --image-project ubuntu-os-cloud `
  --boot-disk-size $DISK_SIZE `
  --tags $TAG
```

### Get VM Public IP

```powershell
gcloud compute instances describe $VM_NAME --zone $ZONE --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

---

## 🔥 PART 2 — Secure Firewall Rules (NO 8080 Public)

### ✅ 2.1 Allow SSH only from your IP

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

### ✅ 2.2 Allow HTTP/HTTPS from Internet (for NGINX + Webhook)

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

### ❌ IMPORTANT

🚫 **Do NOT create any firewall rule for `tcp:8080`**  
Jenkins must stay private behind NGINX.

---

## 🔐 PART 3 — SSH into VM

```powershell
gcloud compute ssh $VM_NAME --zone $ZONE
```

Now you are inside Linux VM 👇

---

## ☕ PART 4 — Install Jenkins + NGINX + Git

```bash
sudo apt update -y
sudo apt install -y fontconfig openjdk-17-jre git nginx
```

### Install Jenkins

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

### Check Jenkins status

```bash
sudo systemctl status jenkins --no-pager
```

---

## 🛡️ PART 5 — Make Jenkins 8080 LOCALHOST ONLY (Secure)

### ✅ Method 1 (Easiest): systemd override file (Recommended)

Create directory:

```bash
sudo mkdir -p /etc/systemd/system/jenkins.service.d
```

Create override file:

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

### Verify Jenkins is localhost-only

```bash
sudo ss -lntp | grep 8080
```

✅ Expected output contains:

```
127.0.0.1:8080
```

---

## 🌐 PART 6 — NGINX Reverse Proxy (Public 80/443 → Jenkins localhost:8080)

Create NGINX config:

```bash
sudo nano /etc/nginx/sites-available/jenkins
```

Paste this:

```nginx
server {
    listen 80;
    server_name jenkins.techwithdiwana.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/jenkins
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

---

## 🔒 PART 7 — Enable HTTPS (SSL)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d jenkins.techwithdiwana.com
```

Now Jenkins should open at:

✅ `https://jenkins.techwithdiwana.com`

---

## 🔑 PART 8 — Jenkins Initial Setup

Get initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Then:

1. Login to Jenkins
2. Install **Recommended Plugins**
3. Create admin user

---

## 🔗 PART 9 — Day-3 Job Creation (Freestyle)

Jenkins → **New Item**

- Name: `day3-git-triggers`
- Type: **Freestyle project**
- Click **OK**

---

## 🧩 PART 10 — GitHub Integration (SCM)

Job → **Configure** → **Source Code Management** → **Git**

Repository URL:

```
https://github.com/techwithdiwana/kubernetes-zero-to-hero.git
```

Branch to build:

```
*/main
```

✅ **Sparse Checkout: OFF** (Day-3 stable demo)

---

## 🔐 PART 11 — Credentials (Recommended)

Jenkins → **Manage Jenkins** → **Credentials** → **Global** → **Add Credentials**

- Kind: **Username with password**
- Username: `techwithdiwana`
- Password: **GitHub PAT**
- ID: `github-creds`

### GitHub PAT permissions (Fine-grained token)

Select minimal permissions:

- ✅ **Contents → Read-only**
- ✅ **Metadata → Read-only** (auto)

Then in Job SCM credentials select:

✅ `github-creds`

---

## ⚡ PART 12 — Build Trigger (Webhook)

Job → Configure → **Build Triggers**

✔ **GitHub hook trigger for GITScm polling**

Save.

---

## 🧪 PART 13 — Build Step (Proof only)

Job → Configure → **Build Steps** → **Execute shell**

Paste:

```bash
echo "🔥 Day-3 Webhook Trigger Working!"
pwd
ls -la
```

Save.

---

## 🪝 PART 14 — GitHub Webhook Setup (IMPORTANT)

GitHub Repo → **Settings** → **Webhooks** → **Add webhook**

### Payload URL

```
https://jenkins.techwithdiwana.com/github-webhook/
```

### Content type

✅ `application/json`

### Secret

Leave **empty** for Day-3

### Events

✅ **Just the push event**

Save webhook.

---

## ✅ PART 15 — Test Webhook (Correct way)

GitHub → Webhook → **Recent Deliveries**

Expected:

- ✅ `ping` event → **200**
- ✅ `push` event → **200**

---

## 🚀 PART 16 — Final Test (Push → Jenkins Auto Trigger)

Make a small change and push:

```bash
git add .
git commit -m "day3 webhook final test"
git push origin main
```

Expected:

- ✅ Jenkins job auto starts
- ✅ Console output shows webhook trigger message

---

## 🧯 PART 17 — If Webhook Not Triggering (Fast Debug)

### 1) Check Jenkins GitHub Hook Log

Jenkins → **GitHub Hook Log**

### 2) Check Firewall Ports

Only these should be public:

- ✅ 80
- ✅ 443

🚫 `8080` should NOT be reachable publicly.

---

## 🏁 Day-3 Completion Criteria (LOCK)

Day-3 is complete only when:

- ✅ Jenkins opens via: `https://jenkins.techwithdiwana.com`
- ✅ `:8080` is not public
- ✅ GitHub webhook push = **200**
- ✅ GitHub push triggers Jenkins job automatically

---

## 🔥 Bonus — Access Jenkins 8080 locally (without exposing it)

From Windows PowerShell:

```powershell
gcloud compute ssh $VM_NAME --zone $ZONE -- -L 8080:127.0.0.1:8080
```

Then open:

✅ `http://localhost:8080`

---

## ⭐ Author

**Tech With Diwana**  
Day 3 — Jenkins Mastery Series

