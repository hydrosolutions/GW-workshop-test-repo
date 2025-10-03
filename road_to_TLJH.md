# The Littlest JupyterHub (TLJH) on AWS EC2

## Complete Deployment Guide

**Author:** Nicolas Lazaro  
**Organization:** Hydrosolutions GmbH  
**Last Updated:** September 2025  
**Purpose:** Workshop deployment guide for groundwater modeling training

---

## Table of Contents

1. [Overview](#overview)
2. [Phase 1: AWS IAM Setup](#phase-1-aws-iam-setup)
3. [Phase 2: Infrastructure Configuration](#phase-2-infrastructure-configuration)
4. [Phase 3: TLJH Installation](#phase-3-tljh-installation)
5. [Phase 4: HTTPS Configuration](#phase-4-https-configuration)
6. [Phase 5: Package Management](#phase-5-package-management)
7. [Production Deployment Summary](#production-deployment-summary)

---

## Overview

This guide documents the complete process for deploying The Littlest JupyterHub on AWS EC2, from IAM user creation through HTTPS configuration. TLJH is ideal for workshops with 1-100 concurrent users.

**Key Features:**

- Single-server JupyterHub deployment
- Shared user environment (no kernel management needed)
- Built-in HTTPS with Let's Encrypt
- Cost-effective for training workshops

---

## Phase 1: AWS IAM Setup

### Prerequisites

- AWS account with root access
- AWS CLI installed locally

### Step 1.1: Create IAM Group and User

Following AWS security best practices, create a dedicated IAM user instead of using root credentials:

**Via AWS Console:**

1. Log into AWS Management Console as root
2. Navigate to **IAM** → **User groups** → **Create group**
   - Group name: `EC2Admins`
   - Attach policy: `AdministratorAccess`
3. Navigate to **IAM** → **Users** → **Create user**
   - Username: `ec2-admin`
   - Add to group: `EC2Admins`
   - Enable **Programmatic access**
4. Download access keys (save securely)

### Step 1.2: Configure AWS CLI Profile

```bash
# Configure new profile
aws configure --profile work
# Enter Access Key ID
# Enter Secret Access Key
# Region: ap-south-1 (Mumbai) or your preferred region
# Output format: json

# Verify credentials
aws sts get-caller-identity --profile work
```

**Expected output:**

```json
{
    "UserId": "AIDAZQSDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/ec2-admin"
}
```

---

## Phase 2: Infrastructure Configuration

### Step 2.1: Create SSH Key Pair

```bash
# Generate key pair
aws ec2 create-key-pair \
  --key-name workshop-central-asia \
  --profile work \
  --region ap-south-1 \
  --query 'KeyMaterial' \
  --output text > ~/Downloads/workshop-central-asia.pem

# Move to secure location and set permissions
mv ~/Downloads/workshop-central-asia.pem ~/.ssh/
chmod 400 ~/.ssh/workshop-central-asia.pem

# Verify
aws ec2 describe-key-pairs \
  --key-names workshop-central-asia \
  --profile work \
  --region ap-south-1
```

### Step 2.2: Configure Security Group

```bash
# Create security group
aws ec2 create-security-group \
  --group-name workshop-sg \
  --description "Security group for JupyterHub workshop" \
  --profile work \
  --region ap-south-1

# Allow SSH (port 22)
aws ec2 authorize-security-group-ingress \
  --group-name workshop-sg \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --profile work \
  --region ap-south-1

# Allow HTTP (port 80)
aws ec2 authorize-security-group-ingress \
  --group-name workshop-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --profile work \
  --region ap-south-1

# Allow HTTPS (port 443)
aws ec2 authorize-security-group-ingress \
  --group-name workshop-sg \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --profile work \
  --region ap-south-1
```

### Step 2.3: Select Ubuntu AMI

```bash
# Find latest Ubuntu 22.04 LTS AMI for your region
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].[ImageId,Name,CreationDate]' \
  --output table \
  --profile work \
  --region ap-south-1
```

**Mumbai region AMI:** `ami-0dee22c13ea7a9a67` (as of Sept 2025)

### Step 2.4: Launch EC2 Instance

**For production workshops (20 users):**

```bash
aws ec2 run-instances \
  --image-id ami-0dee22c13ea7a9a67 \
  --count 1 \
  --instance-type t3.xlarge \
  --key-name workshop-central-asia \
  --security-groups workshop-sg \
  --block-device-mappings DeviceName=/dev/sda1,Ebs='{VolumeSize=20,VolumeType=gp3}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=workshop-central-asia-instance}]' \
  --profile work \
  --region ap-south-1
```

**Instance specs:**

- **Type:** t3.xlarge (4 vCPUs, 16 GB RAM)
- **Storage:** 20 GB gp3 SSD
- **OS:** Ubuntu 22.04 LTS

Save the `InstanceId` from the output (e.g., `i-001c2231d237bb22b`)

### Step 2.5: Allocate and Associate Elastic IP

```bash
# Allocate Elastic IP
aws ec2 allocate-address \
  --domain vpc \
  --profile work \
  --region ap-south-1

# Note the AllocationId and PublicIp from output

# Associate with instance
aws ec2 associate-address \
  --instance-id i-001c2231d237bb22b \
  --allocation-id eipalloc-xxxxxxxxx \
  --profile work \
  --region ap-south-1

# Verify
aws ec2 describe-addresses \
  --profile work \
  --region ap-south-1
```

**Production IP:** `13.204.188.29`

### Step 2.6: Configure DNS

Contact your IT department or domain administrator to create a DNS A record:

**DNS Configuration:**

- **Record Type:** A
- **Host/Name:** jupyter (or hub)
- **Points to:** 13.204.188.29 (your Elastic IP)
- **TTL:** 3600 (or Auto)

This creates: `jupyter.hydrosolutions.ch` → `13.204.188.29`

**Verify DNS propagation:**

```bash
nslookup jupyter.hydrosolutions.ch
# Should return: 13.204.188.29
```

Wait 15 minutes to 2 hours for global DNS propagation.

---

## Phase 3: TLJH Installation

### Step 3.1: SSH into EC2 Instance

```bash
ssh -i ~/.ssh/workshop-central-asia.pem ubuntu@13.204.188.29
```

**First connection:** Type `yes` when prompted about host authenticity.

### Step 3.2: Update System

```bash
# Update package lists and upgrade
sudo apt update && sudo apt upgrade -y

# Verify Python 3
python3 --version
# Should show: Python 3.10.x or newer
```

### Step 3.3: Install TLJH

```bash
# Install TLJH with admin user
curl -L https://tljh.jupyter.org/bootstrap.py | sudo python3 - --admin <your-admin-username>
```

Replace `<your-admin-username>` with your desired admin username (e.g., `nicolas`).

**Installation takes 5-10 minutes** and will:

- Set up Python virtual environment at `/opt/tljh/hub`
- Install JupyterHub and JupyterLab
- Install Traefik reverse proxy
- Create systemd services
- Configure admin user with passwordless sudo

**Completion message:**

```
Done!
```

### Step 3.4: Initial Access Test

Open browser and navigate to:

```
http://jupyter.hydrosolutions.ch
```

You should see the JupyterHub login page. Log in with your admin username and set a password on first login.

---

## Phase 4: HTTPS Configuration

### Step 4.1: Enable HTTPS

While still SSH'd into the server:

```bash
# Enable HTTPS
sudo tljh-config set https.enabled true

# Set Let's Encrypt email for certificate notifications
sudo tljh-config set https.letsencrypt.email your-email@hydrosolutions.ch

# Add your domain (use add-item, not set)
sudo tljh-config add-item https.letsencrypt.domains jupyter.hydrosolutions.ch

# Apply configuration
sudo tljh-config reload proxy
```

**Note:** Use `add-item` because the domains configuration expects an array format.

### Step 4.2: Verify HTTPS

Wait 5-10 minutes for Let's Encrypt to issue the certificate, then access:

```
https://jupyter.hydrosolutions.ch
```

You should see a valid SSL certificate (green padlock) with no browser warnings.

**What happened:**

1. Traefik requested an SSL certificate from Let's Encrypt
2. Let's Encrypt verified domain ownership
3. Certificate issued and installed automatically
4. HTTP traffic now redirects to HTTPS
5. Certificates auto-renew every 90 days

---

## Phase 5: Package Management

### Understanding TLJH's Shared Environment

**Critical Concept:** TLJH uses a shared user environment at `/opt/tljh/user/` that all JupyterHub users access automatically. You install packages once, and all users get them.

**Benefits for workshops:**

- Participants need no technical setup
- No kernel selection required
- Consistent environment across all users
- Simplified instructor management

### Step 5.1: Install Python Packages

**Correct method** (installs to shared environment):

```bash
sudo -E /opt/tljh/user/bin/pip install pandas numpy matplotlib scipy
```

**Common mistake** (installs to system Python, not accessible in notebooks):

```bash
sudo pip install pandas  # ❌ Wrong - don't use this
```

### Step 5.2: Install Conda Packages

For packages better installed via conda:

```bash
sudo -E /opt/tljh/user/bin/conda install -y geopandas rasterio
```

### Step 5.3: Example Workshop Installation

For a groundwater modeling workshop:

```bash
# Install core scientific packages
sudo -E /opt/tljh/user/bin/pip install \
  pandas \
  numpy \
  matplotlib \
  scipy \
  flopy \
  geopandas \
  jupyter \
  ipywidgets

# Enable widgets extension
sudo -E /opt/tljh/user/bin/jupyter nbextension enable --py widgetsnbextension
```

### Step 5.4: Verify Installation

Create a test notebook and run:

```python
import sys
print(f"Python executable: {sys.executable}")
# Should show: /opt/tljh/user/bin/python

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
print("All packages imported successfully!")
```

### Step 5.5: Resource Considerations

**Instance sizing guide:**

- **t3.micro** (1GB RAM): Minimal packages, ~5 users
- **t3.small** (2GB RAM): Basic packages, ~10 users
- **t3.xlarge** (16GB RAM): Full packages, 20+ users

---

## Production Deployment Summary

### Infrastructure Details

**EC2 Instance:**

- **Instance ID:** i-001c2231d237bb22b
- **Type:** t3.xlarge (4 vCPUs, 16 GB RAM)
- **AMI:** Ubuntu 22.04 LTS (ami-0dee22c13ea7a9a67)
- **Storage:** 20 GB gp3 SSD
- **Region:** ap-south-1 (Mumbai)

**Network:**

- **Elastic IP:** 13.204.188.29
- **Domain:** jupyter.hydrosolutions.ch
- **HTTPS:** Enabled with Let's Encrypt

**Access:**

- **SSH:** `ssh -i ~/.ssh/workshop-central-asia.pem ubuntu@13.204.188.29`
- **JupyterHub:** <https://jupyter.hydrosolutions.ch>
- **Admin User:** nicolas

### Cost Estimate

**Monthly costs (approximate):**

- t3.xlarge instance: ~$120/month
- 20 GB gp3 storage: ~$2/month
- Elastic IP: Free while associated
- **Total:** ~$122/month

**Workshop optimization:** Launch instance 1 week before workshop, terminate 1 week after to minimize costs.

---

## Troubleshooting

### SSH Connection Issues

```bash
# If permission denied
chmod 400 ~/.ssh/workshop-central-asia.pem

# If wrong path
ls -la ~/.ssh/workshop-central-asia.pem

# If connection timeout, check security group
aws ec2 describe-security-groups --group-names workshop-sg --profile work --region ap-south-1
```

### HTTPS Certificate Issues

```bash
# Check Traefik logs
sudo journalctl -u traefik -n 50

# Verify domain resolves correctly
nslookup jupyter.hydrosolutions.ch

# Re-apply HTTPS config
sudo tljh-config reload proxy
```

### Package Installation Issues

```bash
# Verify you're using correct pip
which /opt/tljh/user/bin/pip

# Check installation logs
sudo /opt/tljh/user/bin/pip list

# Test in notebook
import sys
print(sys.executable)
```

---

## Maintenance

### Regular Tasks

**Weekly during workshop:**

- Monitor disk usage: `df -h`
- Check memory usage: `free -h`
- Review JupyterHub logs: `sudo journalctl -u jupyterhub -n 100`

**After workshop:**

- Stop instance if not needed: `aws ec2 stop-instances --instance-ids i-001c2231d237bb22b --profile work --region ap-south-1`
- Take snapshot of EBS volume for backup
- Optionally terminate to stop all charges

### Backup Strategy

```bash
# Create AMI of configured instance
aws ec2 create-image \
  --instance-id i-001c2231d237bb22b \
  --name "jupyterhub-workshop-backup-$(date +%Y%m%d)" \
  --description "TLJH configured for workshops" \
  --profile work \
  --region ap-south-1
```

---

## References

- [TLJH Documentation](https://tljh.jupyter.org/)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)

---

## Appendix: Quick Command Reference

```bash
# SSH into server
ssh -i ~/.ssh/workshop-central-asia.pem ubuntu@13.204.188.29

# Install packages (correct method)
sudo -E /opt/tljh/user/bin/pip install package-name

# Restart JupyterHub
sudo systemctl restart jupyterhub

# View JupyterHub logs
sudo journalctl -u jupyterhub -n 50 -f

# Check TLJH configuration
sudo tljh-config show

# Reload proxy (apply HTTPS changes)
sudo tljh-config reload proxy
```

## Phase 6: nbgitpuller with Private Repositories

### Overview

nbgitpuller is a JupyterHub extension that allows you to distribute course materials via simple links. By default, it only works with public repositories. This section covers configuring Git credentials to enable pulling from private GitHub repositories.

### Prerequisites

- Admin access to your organization's GitHub
- TLJH already installed and running
- nbgitpuller installed (typically included with TLJH)

---

### Step 6.1: Generate GitHub Personal Access Token

1. Log into GitHub with your personal account
2. Navigate to **Profile Picture** → **Settings** → **Developer settings**
3. Click **Personal access tokens** → **Tokens (classic)**
4. Click **Generate new token** → **Generate new token (classic)**
5. Configure the token:
   - **Note:** Give it a descriptive name (e.g., `TLJH nbgitpuller - jupyter.hydrosolutions.ch`)
   - **Expiration:** Choose based on your needs (30, 60, 90 days, or custom)
   - **Scopes:** Check **`repo`** (Full control of private repositories)
6. Click **Generate token**
7. **CRITICAL:** Copy the token immediately (format: `ghp_xxxxxxxxxxxx...`)
8. Save it securely - you cannot view it again

**Important Note:** The token is created under your personal account but inherits your permissions to organization repositories.

---

### Step 6.2: Configure Git Credentials on EC2

SSH into your server and configure system-wide Git credentials:

```bash
# SSH into server
ssh -i ~/.ssh/workshop-central-asia.pem ubuntu@13.204.188.29

# Switch to root
sudo su

# Configure git to use credential store for all users
git config --system credential.helper 'store --file /etc/git-credentials'

# Create credentials file (replace with your GitHub username and token)
echo "https://YOUR_GITHUB_USERNAME:ghp_YOUR_TOKEN_HERE@github.com" | sudo tee /etc/git-credentials

# Set permissions so all users can read it
chmod 644 /etc/git-credentials

# Verify the file was created
cat /etc/git-credentials
# Should output: https://YOUR_GITHUB_USERNAME:ghp_YOUR_TOKEN_HERE@github.com

# Exit root
exit
```

**Replace:**

- `YOUR_GITHUB_USERNAME` with your GitHub username (e.g., `CooperBigFoot`)
- `ghp_YOUR_TOKEN_HERE` with the token you generated

**Example:**

```bash
echo "https://CooperBigFoot:XXXX@github.com" | sudo tee /etc/git-credentials
```

---

### Step 6.3: Test Configuration

```bash
# As the ubuntu user (after exiting root)
cd /tmp

# Test that credentials work with your private repository
git ls-remote https://github.com/hydrosolutions/GW-workshop-Zarafshan
```

**Expected output:** List of branches and refs from your private repository:

```
38499cf18040da71cacf7c69bd97da2536c3161c HEAD
38499cf18040da71cacf7c69bd97da2536c3161c refs/heads/main
b97e0b45a0bc0292536e91697c0b6c08fe29ed2e refs/heads/adding_modelling_workflow
...
```

**Note:** You may see a warning `fatal: unable to get credential storage lock in 1000 ms: Permission denied` - this is harmless and can be ignored. The credentials are working correctly.

---

### Step 6.4: Create nbgitpuller Links

Once credentials are configured, create nbgitpuller links for your private repositories:

**General Format:**

```
https://jupyter.hydrosolutions.ch/hub/user-redirect/git-pull?repo=REPO_URL&branch=BRANCH&urlpath=lab/tree/REPO_NAME/PATH_TO_NOTEBOOK
```

**Example for GW-workshop-Zarafshan:**

```
https://jupyter.hydrosolutions.ch/hub/user-redirect/git-pull?repo=https://github.com/hydrosolutions/GW-workshop-Zarafshan&branch=main&urlpath=lab/tree/GW-workshop-Zarafshan/notebooks/1_introduction.ipynb
```

**URL Parameters:**

- `repo`: Full GitHub repository URL
- `branch`: Branch name (usually `main` or `master`)
- `urlpath`: Path within JupyterHub to open after cloning
  - Use `lab/tree/REPO_NAME/path` for JupyterLab interface
  - Use `notebooks/REPO_NAME/path` for classic Notebook interface

**Path Construction:**
The `urlpath` follows this pattern: `lab/tree/{repository-name}/{path-within-repo}`

For example, if your repository is `GW-workshop-Zarafshan` and the notebook is at `notebooks/1_introduction.ipynb`, the urlpath becomes:

```
lab/tree/GW-workshop-Zarafshan/notebooks/1_introduction.ipynb
```

---

### Step 6.5: Share with Workshop Participants

When users click the nbgitpuller link:

1. They'll be prompted to log into JupyterHub (if not already logged in)
2. nbgitpuller automatically clones/pulls the repository to their workspace
3. The specified notebook opens automatically
4. Subsequent clicks will pull updates without overwriting user changes

**Merge Strategy:**
nbgitpuller uses a specialized merge strategy that:

- Pulls new/updated files from the repository
- Preserves user modifications to existing files
- Never causes merge conflicts

This is ideal for workshop materials where instructors update content but participants also modify notebooks.

---

### Security Considerations

**Token Security:**

- Store tokens securely outside of Git repositories
- Use appropriate expiration dates based on workshop schedule
- Revoke tokens after workshops conclude
- Never commit tokens to version control

**File Permissions:**

- `/etc/git-credentials` is readable by all users on the system
- This is necessary for nbgitpuller to function for all workshop participants
- Only admin users (with sudo) can modify the credentials file
- The file is protected from modification by non-admin users

**Best Practices:**

- Create workshop-specific tokens with minimal necessary permissions
- Set expiration dates aligned with workshop duration
- Revoke tokens immediately if compromised
- Use separate tokens for different deployments
- Consider creating a dedicated GitHub account for workshop deployments

**What Users Can Access:**

- All JupyterHub users can clone/pull from repositories using the configured credentials
- Users cannot see the token itself
- Users cannot push changes back to the repository (unless they configure their own credentials)

---

### Troubleshooting

#### Issue: nbgitpuller returns authentication error

```bash
# Verify credentials file exists and has content
cat /etc/git-credentials

# Test credentials manually
git ls-remote https://github.com/your-org/your-private-repo

# Check git configuration
git config --system --list | grep credential
```

**Expected output:**

```
credential.helper=store --file /etc/git-credentials
```

#### Issue: "Permission denied" or "credential storage lock" warning

This warning appears during `git ls-remote` but can be ignored if:

- The repository branches are listed successfully
- nbgitpuller links work correctly

The warning occurs because Git tries to update its credential cache but lacks permissions. The credentials are still used successfully for the operation.

#### Issue: Token expired

**Symptoms:** Authentication failures, "Invalid username or token" errors

**Solution:**

1. Generate a new token following Step 6.1
2. Update `/etc/git-credentials` with the new token:

```bash
sudo su
echo "https://YOUR_USERNAME:ghp_NEW_TOKEN@github.com" | sudo tee /etc/git-credentials
exit
```

3. No restart required; changes take effect immediately

#### Issue: Wrong repository URL in nbgitpuller link

**Symptom:** 404 error or repository not found

**Solution:** Verify the repository URL is exactly correct:

- Include the full URL: `https://github.com/org/repo`
- Check repository name spelling
- Ensure the repository exists and you have access

#### Issue: Notebook path not opening correctly

**Symptom:** nbgitpuller succeeds but opens wrong page

**Solution:** Verify the `urlpath` parameter:

- Must include repository name: `lab/tree/REPO_NAME/path/to/file.ipynb`
- Path is relative to the repository root
- Use forward slashes `/` (not backslashes)

---

### Maintenance

**Before each workshop:**

1. Verify token hasn't expired
2. Test nbgitpuller link with the latest repository content
3. Ensure credentials file is still properly configured

**After workshop:**

1. Consider revoking the token if no longer needed
2. Or extend expiration if planning future workshops

**Token rotation:**
If using the same deployment for multiple workshops, rotate tokens periodically:

1. Generate new token
2. Update `/etc/git-credentials`
3. Test with `git ls-remote`
4. Revoke old token on GitHub

---

### Alternative: Using nbgitpuller Link Generator

For easier link creation, you can use the online nbgitpuller link generator:

**URL:** <https://jupyterhub.github.io/nbgitpuller/link>

**Input:**

1. JupyterHub URL: `https://jupyter.hydrosolutions.ch`
2. Git Repository URL: `https://github.com/hydrosolutions/GW-workshop-Zarafshan`
3. Branch: `main`
4. File to open: `notebooks/1_introduction.ipynb`
5. Application: JupyterLab

The generator will create the properly formatted URL for you.

---

## Quick Reference

```bash
# SSH into server
ssh -i ~/.ssh/workshop-central-asia.pem ubuntu@13.204.188.29

# View configured credentials (as admin)
sudo cat /etc/git-credentials

# Test credentials
git ls-remote https://github.com/your-org/your-private-repo

# Update credentials (as admin)
sudo su
echo "https://USERNAME:TOKEN@github.com" | sudo tee /etc/git-credentials
exit
```

**nbgitpuller URL format:**

```bash
https://jupyter.hydrosolutions.ch/hub/user-redirect/git-pull?repo=REPO_URL&branch=BRANCH&urlpath=lab/tree/REPO_NAME/PATH
```
