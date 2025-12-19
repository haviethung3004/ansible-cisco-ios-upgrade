# Cisco IOS/IOS-XE Upgrade Automation - Ansible Automation Platform Project

An enterprise-grade Ansible project for automating Cisco network device upgrades using **Red Hat Ansible Automation Platform (AAP)**. This project provides a comprehensive, safe, and repeatable workflow for upgrading Cisco IOS and IOS-XE devices with extensive validation, backup, and recovery capabilities.

## Overview

This AAP-integrated project automates the complete lifecycle of Cisco device upgrades:

- **Pre-Upgrade Validation** - Hardware verification, memory checks, image readiness
- **Secure Image Transfer** - SCP-based image delivery with integrity verification
- **Configuration Management** - Automated backup and boot configuration
- **Upgrade Execution** - Orchestrated device reload with rollback capability
- **Post-Upgrade Verification** - System health and version validation

## Project Structure

```
network_project/
├── 01_upgrade_pre_check.yaml         # Pre-upgrade validation and image copy
├── 02_upgrade_bundle_mode.yaml       # Bundle mode upgrade execution
├── 02_upgrade_install_mode.yaml      # Install mode upgrade execution
├── 03_upgrade_post_check.yaml        # Post-upgrade verification
├── playbook.yaml                     # Device information gathering
├── inventory/
│   └── host                          # Inventory definitions
├── collections/
│   ├── requirements.yaml             # Ansible collection dependencies
│   └── ansible_collections/          # Installed collections
├── backup/                           # Timestamped configuration backups
├── images/                           # IOS image repository
└── README.md                         # This documentation
```

## Prerequisites

### Ansible Automation Platform Setup

- **AAP Version:** 2.0+ (Controller)
- **Execution Node:** Python 3.8+, SSH client, sshpass
- **Collections:** Automatically installed from `collections/requirements.yaml`
- **Network:** SSH (port 22) and SCP connectivity to Cisco devices

### Required Ansible Collections

```yaml
# collections/requirements.yaml
collections:
  - cisco.ios
```

These are automatically installed when the project is synced in AAP Controller.

### Network Access Requirements

- SSH connectivity to all Cisco devices (TCP 22)
- SCP connectivity to image repository (TCP 22)
- Network connectivity between AAP execution nodes and devices
- Minimum 2GB free flash memory on target devices

## Configuration in Ansible Automation Platform

### 1. Create/Update Project

1. Navigate to **Administration > Projects**
2. Click **Create project**
3. Configure:
   - **Name:** `Cisco IOS Upgrade`
   - **Organization:** Your organization
   - **Execution Environment:** Default (or custom with required collections)
   - **Source Control Type:** Git
   - **Source Control URL:** Your repository URL
   - **Source Control Branch:** `main`

### 2. Create Inventory

1. Navigate to **Resources > Inventories**
2. Click **Create inventory**
3. Configure:
   - **Name:** `Cisco Network Devices`
   - **Organization:** Your organization
   - **Execution Environment:** Default

4. Add hosts to inventory - navigate to **Hosts** and add:

```ini
[cisco_ios_devices]
router1 ansible_host=192.168.1.1
switch1 ansible_host=192.168.1.2
switch2 ansible_host=192.168.1.3

[cisco_ios_devices:vars]
ansible_network_os=cisco.ios.ios
ansible_connection=network_cli
ansible_become=yes
ansible_become_method=enable
ansible_command_timeout=10800
```

### 3. Create Credentials

#### Device Machine Credential

1. Navigate to **Resources > Credentials**
2. Click **Create credential**
3. Configure:
   - **Name:** `Cisco Device Credentials`
   - **Credential Type:** `Machine`
   - **Username:** `admin` (or your device username)
   - **Password:** Your device password
   - **Privilege Escalation Password:** Your enable password (if required)

#### SCP Server Credential (Optional)

Create a Custom Credential Type for SCP variables:

1. Navigate to **Administration > Credential Types**
2. Click **Create credential type**
3. Input Configuration:
   ```yaml
   fields:
     - id: scp_server
       type: string
       label: SCP Server IP/Hostname
     - id: scp_username
       type: string
       label: SCP Username
     - id: scp_password
       type: string
       label: SCP Password
       secret: true
   ```
4. Injector Configuration:
   ```yaml
   extra_vars:
     scp_server: '{{ scp_server }}'
     scp_username: '{{ scp_username }}'
     scp_password: '{{ scp_password }}'
   ```

### 4. Define Extra Variables

Define these variables for job launches (via survey or extra vars):

| Variable | Type | Required | Example |
|----------|------|----------|---------|
| `scp_server` | string | Yes | `10.0.0.100` |
| `scp_username` | string | Yes | `upload_user` |
| `scp_password` | string | Yes | `SecurePass123` |
| `new_ios_version` | string | Yes | `17.9.4a` |
| `new_image_name` | string | Yes | `c9300-universalk9.17.09.04a.SPA.bin` |
| `new_image_md5` | string | Yes | `a1b2c3d4e5f6...` |

## Playbooks

### 1. Pre-Upgrade Check (`01_upgrade_pre_check.yaml`)

**Purpose:** Validates device readiness, transfers image, and verifies integrity

**Key Tasks:**
- Gathers hardware facts and current configuration
- Validates current IOS version (prevents redundant upgrades)
- Checks available flash memory (minimum 2GB required)
- Verifies/sets configuration register (0x2102 for boot from flash)
- Transfers new IOS image via SCP (if not already present)
- Validates MD5 checksum against provided hash
- Generates comprehensive pre-upgrade report

**AAP Job Template Setup:**

1. Navigate to **Resources > Templates > Create job template**
2. Configure:
   - **Name:** `Cisco IOS - Pre-Upgrade Check`
   - **Playbook:** `01_upgrade_pre_check.yaml`
   - **Inventory:** `Cisco Network Devices`
   - **Credentials:** `Cisco Device Credentials`
   - **Execution Environment:** Default (or custom)
   - **Extra Variables:** (see survey below)
   - **Verbosity:** 2 (Verbose)

3. Create Survey for user-friendly variable input:
   ```
   Question 1: SCP Server IP (text)
   Question 2: SCP Username (text)
   Question 3: SCP Password (password)
   Question 4: New IOS Version (text)
   Question 5: New Image Name (text)
   Question 6: New Image MD5 (text)
   ```

### 2. Upgrade Execution - Bundle Mode (`02_upgrade_bundle_mode.yaml`)

**Purpose:** Traditional unified image upgrade method

**Key Tasks:**
- Creates timestamped backup of running configuration
- Transfers backup to SCP server
- Sets boot system variable to new image
- Saves configuration changes
- Prepares device for reload

**AAP Job Template Setup:**

1. **Name:** `Cisco IOS - Upgrade (Bundle Mode)`
2. **Playbook:** `02_upgrade_bundle_mode.yaml`
3. **Dependencies:** Execute after successful pre-upgrade check
4. **Survey Variables:**
   - SCP Server IP
   - SCP Username
   - SCP Password
   - New Image Name
   - Backup Reason (optional)

### 3. Upgrade Execution - Install Mode (`02_upgrade_install_mode.yaml`)

**Purpose:** Modern install mode upgrade (IOS-XE 16.12+)

**Key Tasks:**
- Prepares device for install mode operation
- Manages ROMMON fallback image
- Executes install add/commit workflow
- Validates new package installation
- Performs automated reload if configured

**AAP Job Template Setup:**

1. **Name:** `Cisco IOS - Upgrade (Install Mode)`
2. **Playbook:** `02_upgrade_install_mode.yaml`
3. **Note:** For IOS-XE 16.12 and later only

### 4. Post-Upgrade Verification (`03_upgrade_post_check.yaml`)

**Purpose:** Validates successful upgrade and system health

**Key Tasks:**
- Verifies new IOS version is running
- Checks device stability metrics
- Validates interface status
- Confirms routing protocol operation
- Generates comprehensive post-upgrade report
- (To be fully implemented)

**AAP Job Template Setup:**

1. **Name:** `Cisco IOS - Post-Upgrade Check`
2. **Playbook:** `03_upgrade_post_check.yaml`
3. **Execute:** After device reload (manual or scheduled)

### 5. Device Information Gathering (`playbook.yaml`)

**Purpose:** Collects comprehensive device inventory and facts

**Key Tasks:**
- Gathers detailed hardware facts
- Collects software version information
- Displays device model, serial number, uptime
- Shows running configuration
- Lists interface status and VLAN information
- Generates device inventory report

**AAP Job Template Setup:**

1. **Name:** `Cisco Device Information`
2. **Playbook:** `playbook.yaml`
3. **Inventory:** `Cisco Network Devices`
4. **Purpose:** Pre-upgrade data collection for change documentation

## Recommended Workflow in Ansible Automation Platform

### Pre-Upgrade Phase

1. **Create Change Request** - Document upgrade plan with business justification
2. **Run Device Information Job** - Collect baseline inventory and facts
   - Template: `Cisco Device Information`
   - Purpose: Establish pre-upgrade state for comparison
   - Output: Screenshot/export for change documentation

3. **Execute Pre-Upgrade Check** - Validate readiness and prepare image
   - Template: `Cisco IOS - Pre-Upgrade Check`
   - Input: SCP server details, target image version, MD5 hash
   - Verify: Image copied successfully, MD5 validated, memory sufficient

### Upgrade Phase

4. **Run Upgrade Execution** - Back up configuration and prepare for reload
   - Template: `Cisco IOS - Upgrade (Bundle Mode)` or `(Install Mode)`
   - Input: Same SCP credentials, image name
   - Result: Configuration backed up, boot system configured

5. **Manual Device Reload** (if not automated)
   - SSH to device and execute: `reload in 5`
   - Monitor device during boot process
   - Confirm reload completion (5-15 minutes typical)

### Post-Upgrade Phase

6. **Run Post-Upgrade Check** - Validate successful upgrade
   - Template: `Cisco IOS - Post-Upgrade Check`
   - Wait time: 5 minutes after device reload
   - Verify: Correct version running, interfaces up, protocols converged

7. **Collect Post-Upgrade Data** - Document successful state
   - Template: `Cisco Device Information`
   - Purpose: Compare with pre-upgrade baseline
   - Document: Verify expected configuration retained

8. **Close Change Request** - Update tracking with completion status

### Key Workflow Features

- **Job Scheduling:** AAP supports scheduled upgrades during maintenance windows
- **Approval Gates:** Configure workflow approval before each phase
- **Notifications:** Email/Slack alerts on job completion/failure
- **RBAC:** Role-based access control for operator authorization
- **Audit Trail:** Complete job history and execution logs for compliance

## Enterprise Features

### Safety Mechanisms

- **Version Detection:** Prevents upgrade if target version already running
- **Memory Validation:** Enforces 2GB minimum free flash space requirement
- **Image Integrity:** Mandatory MD5 checksum validation
- **Configuration Protection:** Automatic timestamped backup before changes
- **Idempotent Checks:** Image copy skipped if already present on device
- **Boot Register Validation:** Ensures correct ROMMON boot configuration
- **Error Handling:** Comprehensive error recovery and rollback capability

### Scalability & Operations

- **Multi-Device Upgrade:** Supports orchestrated upgrades across device groups
- **Parallelization:** Configure job template for serial or batch execution
- **Progress Tracking:** Real-time job output streaming in AAP UI
- **Execution Environments:** Custom EE support for specialized requirements
- **Logging & Audit:** Complete execution logs for compliance and troubleshooting
- **Notification Integration:** Slack/Email alerts for operational awareness

### Troubleshooting Guide

| Issue | Root Cause | Solution |
|-------|-----------|----------|
| Insufficient memory error | Flash space < 2GB | Use `delete flash:` to remove old images |
| SCP transfer timeout | Network connectivity | Verify SCP server reachability, check firewall rules |
| MD5 verification failed | Image corruption | Re-download image from Cisco, verify checksum |
| Device SSH unreachable | Connectivity/credentials | Verify `ansible_host`, username, password, enable secret |
| Image not found after copy | Interrupted transfer | Check device connectivity, retry with increased timeout |
| Configuration register mismatch | Boot configuration issue | Manually verify `show version \| i register` output |
| Reload failed | ROMMON issues | Requires console access for manual intervention |

### Network Requirements Checklist

- [ ] SSH access to all devices on TCP 22
- [ ] SCP server accessibility (TCP 22)
- [ ] Execution node network connectivity verified
- [ ] Firewall rules allow AAP-to-device communication
- [ ] Each device has minimum 2GB free flash space
- [ ] Device enable passwords configured (if RBAC in use)
- [ ] NTP synchronized for accurate logging (recommended)
- [ ] Console access available (for emergency recovery)

## Support & Compatibility

**Tested On:**
- Cisco Catalyst 9300 Series
- Cisco ISR 4000 Series
- Cisco ASR Series
- IOS-XE 16.x, 17.x, and later

**Platform Requirements:**
- **AAP Version:** 2.0+ (formerly Ansible Tower)
- **Ansible Version:** 2.9+
- **Collection Versions:** cisco.ios 4.0+
- **Python:** 3.8+ on execution nodes

**Network Support:**
- IPv4 device management
- SSH-based access (network_cli connection)
- Cisco network_os drivers

## Additional Resources

- [Cisco IOS Documentation](https://www.cisco.com/c/en/us/support/ios-nx-os-software/ios-software/products.html)
- [Ansible Automation Platform Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/)
- [Ansible Cisco iOS Collection](https://github.com/ansible-collections/cisco.ios)
- [Network Best Practices Guide](https://docs.ansible.com/ansible/latest/network/index.html)

## Contributing

This project is designed to be extended and customized for specific environments:

- **Submit Issues:** Report bugs or feature requests via project issue tracker
- **Contribute:** Submit pull requests for improvements, bug fixes, or new features
- **Feedback:** Share deployment experiences and recommendations
- **Documentation:** Help improve playbook documentation and examples

## License

This project is provided as-is for enterprise network automation purposes. Modify and distribute according to your organization's policies.

## Author & Support

**Network Automation Team**

For support, questions, or customization needs, contact your network operations team or infrastructure automation group.

---

**Project Status:** Active Development  
**Last Updated:** December 2025  
**Version:** 2.0  
**Ansible Automation Platform Certified:** Yes
