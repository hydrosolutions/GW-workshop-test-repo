# JupyterHub Setup Plan for Groundwater Modeling Workshop

## Objective

Set up The Littlest JupyterHub (TLJH) on AWS EC2 to deliver a groundwater modeling workshop for 20 young professionals in Tajikistan with minimal technical barriers. Participants will access the environment through a single link that automatically opens JupyterLab with the first notebook ready, requiring only basic login credentials.

### Key Requirements

- **Participants**: 20 environmental engineers with limited coding experience
- **Location**: Tajikistan (2 months from now)
- **User Experience**: One-click access via nbgitpuller link
- **Interface**: JupyterLab with pre-loaded notebooks and data
- **Technical Level**: Minimal - some participants barely use Excel

---

## Phase 1: Content Preparation

### Step 1: Create GitHub Repository

- Create public GitHub repository for workshop materials
- Structure: Keep repository lean for fast initial pulls

  ```text
  groundwater-workshop/
  ├── notebooks/
  │   ├── 01-Introduction.ipynb
  │   ├── 02-DataPreparation.ipynb
  │   └── ...
  ├── data/
  │   └── small_datasets_only/
  ├── environment.yml
  └── README.md
  ```

- Include `environment.yml` with all required packages
- Push all materials to main branch

---

## Phase 2: AWS Infrastructure Setup

### Step 3: AWS Region and Instance Selection

- **Region**: Choose closest to Tajikistan for lower latency
  - Primary: Mumbai (`ap-south-1`)
  - Alternatives: UAE (`me-central-1`), Bahrain (`me-south-1`)
- **Instance Type**:
  - Start with `t3.large` (2 vCPU, 8 GiB RAM) for light use
  - Or `t3.xlarge` (4 vCPU, 16 GiB RAM) for heavy numerical work
  - Can resize later if needed

### Step 4: Launch EC2 Instance

- **AMI**: Ubuntu Server 22.04 LTS
- **Storage**: 100-200 GB EBS volume (increase if hosting large datasets)
- **Security Group Configuration**:
  - SSH (port 22): Restricted to your office IP addresses
  - HTTP (port 80): Open to all (0.0.0.0/0)
  - HTTPS (port 443): Open to all (0.0.0.0/0)
- **Key Pair**: Create and download for SSH access

### Step 5: Configure Networking

- Allocate and associate an Elastic IP to the instance
- (Optional but recommended) Configure DNS:
  - Point a domain/subdomain (e.g., `hub.yourdomain.org`) to the Elastic IP
  - Creates professional appearance and enables easy HTTPS setup

---

## Phase 3: JupyterHub Installation

### Step 6: Install The Littlest JupyterHub

- SSH into instance as `ubuntu` user
- Run TLJH bootstrap script:

  ```bash
  curl -L https://tljh.jupyter.org/bootstrap.py | sudo python3 - --admin <your-admin-username>
  ```

- Wait for installation to complete (~5-10 minutes)

### Step 7: Enable HTTPS

- Use TLJH's built-in Let's Encrypt integration:

  ```bash
  sudo tljh-config set https.enabled true
  sudo tljh-config set https.letsencrypt.email your-email@domain.com
  sudo tljh-config add-item https.letsencrypt.domains hub.yourdomain.org
  sudo tljh-config reload proxy
  ```

### Step 8: Configure Authentication

- Use **FirstUseAuthenticator** (default) for simplicity:

  ```bash
  sudo tljh-config set auth.type firstuseauthenticator.FirstUseAuthenticator
  sudo tljh-config set auth.FirstUseAuthenticator.create_users true
  sudo tljh-config reload hub
  ```

- Pre-create user accounts for workshop control

---

## Phase 4: Environment Configuration

*Timeline: 2-3 weeks before workshop*

### Step 9: Install Required Packages

- Access JupyterHub as admin
- Open terminal in JupyterLab
- Install packages for all users:

  ```bash
  # Core scientific packages
  sudo -E pip install numpy pandas matplotlib scipy
  
  # Groundwater modeling specific packages
  sudo -E pip install flopy modflow-exe
  
  # Or use conda for complex dependencies
  sudo -E conda install -c conda-forge <package-name>
  ```

### Step 10: Install nbgitpuller

- Install nbgitpuller extension:

  ```bash
  sudo -E pip install nbgitpuller
  sudo -E jupyter serverextension enable --py nbgitpuller --sys-prefix
  ```

### Step 11: Configure JupyterLab as Default

- Set JupyterLab as default interface:

  ```bash
  sudo tljh-config set user_environment.default_app jupyterlab
  sudo tljh-config reload hub
  ```

### Step 12: Set Resource Limits

- Configure per-user resource limits:

  ```bash
  # Memory limit per user (e.g., 2GB)
  sudo tljh-config set limits.memory 2G
  # CPU limit (e.g., 1 core)
  sudo tljh-config set limits.cpu 1
  sudo tljh-config reload hub
  ```

---

## Phase 5: Content Delivery Setup

*Timeline: 1-2 weeks before workshop*

### Step 13: Prepare Large Datasets (if needed)

- For datasets too large for Git:
  - Upload directly to server via SCP/SFTP
  - Place in shared directory (e.g., `/srv/data`)
  - Create read-only symlinks to user directories

### Step 14: Generate nbgitpuller Link

- Visit: <https://nbgitpuller.link>
- Fill in:
  - **JupyterHub URL**: `https://hub.yourdomain.org`
  - **Repository URL**: Your GitHub repo URL
  - **Branch**: `main` or `master`
  - **Path to open**: `/notebooks/01-Introduction.ipynb`
  - **Application**: JupyterLab
- Generate and save the link
- Consider using URL shortener (e.g., bit.ly) for easier sharing

### Step 15: Create User Accounts

- As admin, create accounts for all 20 participants:

  ```bash
  # Can script this or do via admin panel
  sudo tljh-config add-item users.allowed user01
  sudo tljh-config add-item users.allowed user02
  # ... continue for all users
  sudo tljh-config reload hub
  ```

- Prepare simple username/password combinations
- Document in a spreadsheet for distribution

---

## Phase 6: Testing and Validation

*Timeline: 1 week before workshop*

### Step 16: End-to-End Testing

- Test with multiple dummy accounts simultaneously
- Verify workflow:
  1. Click nbgitpuller link
  2. Login with credentials
  3. Repository clones automatically
  4. First notebook opens in JupyterLab
  5. All cells execute without errors
  6. Can save work and re-pull updates

### Step 17: Stress Testing

- Simulate 20 concurrent users
- Monitor system resources:

  ```bash
  htop  # CPU and memory usage
  df -h  # Disk usage
  ```

- Resize instance if performance issues

### Step 18: Create Backups

- Create AMI snapshot of configured instance
- Export notebooks to local backup
- Document recovery procedures

---

## Phase 7: Workshop Preparation

*Timeline: 2-3 days before workshop*

### Step 19: Prepare Participant Materials

Create simple one-page handout with:

- Workshop URL (the nbgitpuller link)
- Individual username and password
- Basic troubleshooting:
  - Browser requirements (Chrome/Firefox recommended)
  - Clear cache if issues
  - Refresh page if notebook doesn't load

### Step 20: Prepare Instructor Resources

- Admin dashboard URL
- SSH access commands
- How to restart user servers
- How to monitor system resources
- Emergency contact for AWS support

### Step 21: Create Fallback Plan

- USB drives with:
  - Anaconda installers
  - Workshop notebooks
  - Required datasets
- Test local Jupyter installation on instructor laptops
- Optional: Prepare Binder deployment as online backup

---

## Phase 8: Day of Workshop

### Step 22: Pre-Workshop Checklist

- [ ] EC2 instance running
- [ ] HTTPS certificate valid (green lock)
- [ ] Admin account accessible
- [ ] Test student account works end-to-end
- [ ] nbgitpuller link opens correct notebook
- [ ] System load acceptable
- [ ] Internet connectivity stable
- [ ] Backup plan materials ready

### Step 23: Workshop Delivery

- Distribute handouts with credentials
- Share nbgitpuller link (project on screen)
- Have participants test login immediately
- Monitor system performance
- Be ready to restart individual user servers if needed

### Step 24: Real-time Management

- Keep admin panel open for user management
- Monitor via SSH terminal:

  ```bash
  # Watch system resources
  watch -n 5 'free -h; df -h'
  ```

- Document any issues for future reference

---

## Phase 9: Post-Workshop

### Step 25: Cleanup and Cost Management

- **Option A**: Stop instance (preserves data, minimal cost)

  ```
  AWS Console → EC2 → Instances → Stop
  ```

- **Option B**: Create final snapshot then terminate
- Download any participant work if needed
- Release Elastic IP if not needed

### Step 26: Documentation

- Document lessons learned
- Save configuration scripts
- Update this plan based on experience
- Share feedback with participants

---

## Critical Success Factors

### Network Considerations for Tajikistan

- Test bandwidth requirements in advance
- Consider pre-loading all data on server
- Minimize real-time downloads during workshop
- Have offline alternatives ready

### Cost Optimization

- **Estimated costs**: ~$50-150 for workshop duration
- Stop instance when not actively testing
- Use AWS Free Tier if eligible
- Consider Reserved Instance for multiple workshops

### Key Commands Reference

```bash
# Check TLJH status
sudo systemctl status jupyterhub

# View logs
sudo journalctl -u jupyterhub

# Restart hub
sudo tljh-config reload hub

# Add admin user
sudo tljh-config add-item users.admin <username>

# Update packages
sudo -E pip install --upgrade <package>
```

### Support Resources

- TLJH Documentation: <https://tljh.jupyter.org>
- nbgitpuller Docs: <https://nbgitpuller.readthedocs.io>
- AWS EC2 Guide: <https://docs.aws.amazon.com/ec2/>
- JupyterHub Community Forum: <https://discourse.jupyter.org>

---

## Final Notes

This plan prioritizes simplicity and reliability for non-technical users. The single nbgitpuller link approach eliminates most technical barriers, allowing participants to focus on learning groundwater modeling rather than struggling with setup. Remember that for a workshop in Tajikistan, having robust offline alternatives is essential given potential connectivity challenges.
