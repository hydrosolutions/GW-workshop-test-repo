# Road to The Littlest JupyterHub (TLJH) - AWS Setup Documentation

## Overview

This document outlines the complete AWS infrastructure setup for hosting The Littlest JupyterHub (TLJH) on EC2, including IAM user creation, EC2 instance configuration, and successful TLJH installation.

## Phase 1: IAM Setup and User Creation ✅ COMPLETED

### Creating the EC2Admin Group and User

To follow AWS security best practices, we avoided using the root account for day-to-day operations and instead created a dedicated IAM user with administrative privileges.

**Manual Steps via AWS Console:**

1. Logged into AWS Management Console as root user
2. Navigated to IAM service
3. Created IAM group named `EC2Admins` with `AdministratorAccess` policy attached
4. Created IAM user `ec2-admin` and added it to the `EC2Admins` group
5. Generated access keys for programmatic access
6. Downloaded credentials

**Initial CLI Configuration:**

```bash
aws configure --profile ec2-admin
# Entered Access Key ID, Secret Access Key
# Set region to eu-central-2 (later changed to eu-central-1)
# Set output format to json
```

**Troubleshooting Credentials:**
Initially encountered `InvalidClientTokenId` error. Resolved by updating credentials:

```bash
aws configure set aws_access_key_id AKIAZQSDA66HN5AR767O --profile ec2-admin
aws configure set aws_secret_access_key 'SECRET_KEY_HERE' --profile ec2-admin
```

**Region Adjustment:**
Changed region from eu-central-2 to eu-central-1 (Frankfurt) for better service availability:

```bash
aws configure set region eu-central-1 --profile ec2-admin
```

**Verification:**

```bash
aws sts get-caller-identity --profile ec2-admin
# Successfully returned: UserId, Account, and Arn confirming access
```

## Phase 2: AWS Infrastructure Setup ✅ COMPLETED

### Step 1: SSH Key Pair Creation

Created an SSH key pair for secure access to the EC2 instance:

```bash
# Create key pair and save private key locally
aws ec2 create-key-pair --key-name littles-hub-key --profile ec2-admin --output text --query 'KeyMaterial' > ~/.ssh/littles-hub-key.pem

# Set proper permissions for SSH key
chmod 400 ~/.ssh/littles-hub-key.pem

# Verify key pair creation
aws ec2 describe-key-pairs --key-names littles-hub-key --profile ec2-admin --output table
```

### Step 2: Security Group Configuration

Created a security group to control network access to the JupyterHub instance:

```bash
# Create security group
aws ec2 create-security-group --group-name littles-hub-sg --description "Security group for The Littles JupyterHub" --profile ec2-admin

# Allow SSH access (port 22) from anywhere
aws ec2 authorize-security-group-ingress --group-name littles-hub-sg --protocol tcp --port 22 --cidr 0.0.0.0/0 --profile ec2-admin

# Allow HTTP access (port 80) from anywhere
aws ec2 authorize-security-group-ingress --group-name littles-hub-sg --protocol tcp --port 80 --cidr 0.0.0.0/0 --profile ec2-admin

# Allow HTTPS access (port 443) from anywhere
aws ec2 authorize-security-group-ingress --group-name littles-hub-sg --protocol tcp --port 443 --cidr 0.0.0.0/0 --profile ec2-admin

# Verify security group configuration
aws ec2 describe-security-groups --group-names littles-hub-sg --profile ec2-admin --output table
```

### Step 3: AMI Selection

Found the appropriate Ubuntu 22.04 LTS AMI for the Frankfurt region:

```bash
# Find latest Ubuntu 22.04 LTS AMI
aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" --query 'Images[0].[ImageId,Name]' --output table --profile ec2-admin
# Result: ami-01cd11f529b248fd4
```

### Step 4: EC2 Instance Launch

Initially attempted to use t3.large but switched to t3.micro for free tier eligibility:

```bash
# Check free tier eligible instance types
aws ec2 describe-instance-types --filters "Name=free-tier-eligible,Values=true" --query 'InstanceTypes[*].[InstanceType,VCpuInfo.DefaultVCpus,MemoryInfo.SizeInMiB]' --output table --profile ec2-admin

# Launch EC2 instance with t3.micro (free tier)
aws ec2 run-instances --image-id ami-01cd11f529b248fd4 --count 1 --instance-type t3.micro --key-name littles-hub-key --security-groups littles-hub-sg --block-device-mappings DeviceName=/dev/sda1,Ebs='{VolumeSize=20,VolumeType=gp3}' --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=littles-jupyterhub}]' --profile ec2-admin
```

### Step 5: Elastic IP Configuration

Allocated and associated an Elastic IP for consistent network access:

```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc --profile ec2-admin
# Result: AllocationId: eipalloc-0a4ce318784404a2c, PublicIp: 63.179.186.206

# Associate Elastic IP with instance
aws ec2 associate-address --instance-id i-04ed29cb6be12bae0 --allocation-id eipalloc-0a4ce318784404a2c --profile ec2-admin

# Verify Elastic IP association
aws ec2 describe-instances --instance-ids i-04ed29cb6be12bae0 --query 'Reservations[*].Instances[*].[PublicIpAddress,PublicDnsName]' --output table --profile ec2-admin
```

## Phase 3: JupyterHub Installation ✅ COMPLETED

### Step 6: SSH Connection and System Preparation

Successfully connected to the EC2 instance and prepared the system:

```bash
# SSH into the instance
ssh -i ~/.ssh/littles-hub-key.pem ubuntu@63.179.186.206

# Update system packages
sudo apt update && sudo apt upgrade -y

# Verify Python 3 (returned 3.10.12)
python3 --version

# Verify curl installation
curl --version
```

### Step 7: TLJH Installation

Successfully installed The Littlest JupyterHub using the bootstrap script:

```bash
# Install TLJH with admin user
curl -L https://tljh.jupyter.org/bootstrap.py | sudo python3 - --admin myuser
```

**Installation completed successfully with:**

- Hub environment set up with virtual environment at `/opt/tljh/hub`
- Admin user "myuser" created with passwordless sudo access
- User environment with Miniforge3 conda installed
- JupyterHub and JupyterLab configured
- Traefik proxy (v3.1.4) set up for web server
- SystemD services created and started:
  - `jupyterhub.service`
  - `traefik.service`

### Step 8: Access Verification ✅ SUCCESS

**JupyterHub is now accessible at:** `http://63.179.186.206`

- Login page loads successfully
- Admin user "myuser" can log in (password set on first login)
- JupyterLab interface accessible after login

## Current Infrastructure Summary

**EC2 Instance Details:**

- **Instance ID:** i-04ed29cb6be12bae0
- **Instance Type:** t3.micro (2 vCPUs, 1GB RAM)
- **AMI:** Ubuntu Server 22.04 LTS (ami-01cd11f529b248fd4)
- **Storage:** 20GB EBS gp3 volume
- **Region:** eu-central-1 (Frankfurt)
- **Availability Zone:** eu-central-1b

**Network Configuration:**

- **Elastic IP:** 63.179.186.206
- **Public DNS:** ec2-63-179-186-206.eu-central-1.compute.amazonaws.com
- **Security Group:** littles-hub-sg
  - SSH (22): Open to 0.0.0.0/0
  - HTTP (80): Open to 0.0.0.0/0
  - HTTPS (443): Open to 0.0.0.0/0

**TLJH Configuration:**

- **Access URL:** <http://63.179.186.206>
- **Admin User:** myuser
- **Default Interface:** JupyterLab
- **Conda Environment:** Miniforge3 (24.7.1-2)
- **Proxy:** Traefik 3.1.4

**Access Configuration:**

- **SSH Key:** littles-hub-key (stored at ~/.ssh/littles-hub-key.pem)
- **IAM User:** ec2-admin (AdministratorAccess)

## Next Steps for Workshop Preparation

Now that TLJH is successfully installed and accessible, the following phases remain:

### Phase 4: Environment Configuration (TODO)

- Install required packages for groundwater modeling (flopy, modflow-exe, etc.)
- Install nbgitpuller extension
- Configure JupyterLab as default interface
- Set per-user resource limits

### Phase 5: Content Delivery Setup (TODO)

- Create GitHub repository with workshop materials
- Generate nbgitpuller link for one-click access
- Create user accounts for 20 participants
- Test end-to-end workflow

### Phase 6: HTTPS Setup (TODO)

- Configure Let's Encrypt SSL certificate for secure access
- Update access URL to https://

### Phase 7: Testing and Validation (TODO)

- Stress test with multiple concurrent users
- Create backups and recovery procedures
- Prepare fallback options

## Key Learnings and Success Factors

**What worked well:**

- Free tier t3.micro instance sufficient for basic TLJH installation
- Ubuntu 22.04 LTS provided stable base with Python 3.10.12
- TLJH bootstrap script handled all complex configuration automatically
- Elastic IP ensured consistent access throughout setup

**Critical commands for TLJH management:**

```bash
# Check TLJH status
sudo systemctl status jupyterhub

# View logs
sudo journalctl -u jupyterhub

# Restart hub
sudo tljh-config reload hub

# Add admin user
sudo tljh-config add-item users.admin <username>
```

**Current Status:** TLJH successfully installed and accessible. Ready for workshop-specific configuration and content preparation.

## Resources

- TLJH Documentation: <https://tljh.jupyter.org>
- AWS EC2 Guide: <https://docs.aws.amazon.com/ec2/>
- JupyterHub Community Forum: <https://discourse.jupyter.org>
