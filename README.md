# Cisco IOS/IOS-XE Upgrade Automation Project

This Ansible project automates the process of upgrading Cisco IOS and IOS-XE devices with comprehensive pre-checks, upgrade execution, and post-verification steps.

## Overview

This project provides a complete workflow for safely upgrading Cisco network devices including:
- Pre-upgrade checks and validations
- Automated image transfer via SCP
- MD5 verification
- Configuration backup
- Safe upgrade execution
- Post-upgrade verification

## Project Structure

```
network_project/
├── 01_upgrade_pre_check.yaml    # Pre-upgrade validation and image copy
├── 02_upgrade.yaml              # Backup and upgrade execution
├── 03_upgrade_post_check.yaml   # Post-upgrade verification
├── playbook.yaml                # Device information gathering
├── inventory/
│   └── host                     # Inventory file for target devices
├── collections/
│   ├── requirements.yaml        # Ansible collection dependencies
│   └── ansible_collections/
├── backup/                      # Configuration backups
└── images/                      # IOS images storage
```

## Prerequisites

### Required Software
- AWX (Ansible Tower) installed and configured
- Python 3.6+ on AWX execution nodes
- sshpass (for SCP transfers) on AWX execution nodes

### Ansible Collections
- cisco.ios

Collections are automatically installed from `collections/requirements.yaml` when the project is synced in AWX.

### Network Requirements
- SSH access to Cisco devices
- SCP server for image hosting
- Network connectivity between Ansible controller and devices

## Configuration

### Inventory Setup in AWX

1. **Create Inventory** in AWX:
   - Name: `Cisco Network Devices`
   
2. **Add Hosts** to inventory:
   ```ini
   [cisco_devices]
   router1 ansible_host=192.168.1.1
   switch1 ansible_host=192.168.1.2
   
   [cisco_devices:vars]
   ansible_network_os=cisco.ios.ios
   ansible_connection=network_cli
   ansible_become=yes
   ansible_become_method=enable
   ```

3. **Store Credentials** in AWX:
   - Create Machine Credential for device access
   - Username: admin
   - Password: (device password)

### Required Variables

Define these variables in AWX as **Extra Variables** when launching Job Templates:

| Variable | Description | Example |
|----------|-------------|---------|
| `scp_server` | SCP server IP/hostname | `10.0.0.100` |
| `scp_username` | SCP server username | `admin` |
| `scp_password` | SCP server password | `SecurePass123` |
| `new_ios_version` | Target IOS version | `17.9.4a` |
| `new_image_name` | IOS image filename | `c9300-universalk9.17.09.04a.SPA.bin` |
| `new_image_md5` | MD5 checksum of image | `a1b2c3d4e5f6...` |

## Playbooks

### 1. Pre-Upgrade Check (`01_upgrade_pre_check.yaml`)

**Purpose:** Validates device readiness and prepares for upgrade

**Tasks:**
- Gathers hardware facts
- Checks current IOS version
- Validates available flash memory (minimum 2GB required)
- Verifies configuration register (sets to 0x2102 if needed)
- Checks if new image already exists on device
- Copies new IOS image via SCP (if not present)
- Verifies MD5 checksum of image

**AWX Setup:**
1. Create Job Template: `Cisco IOS Pre-Upgrade Check`
2. Playbook: `01_upgrade_pre_check.yaml`
3. Inventory: Select your Cisco devices inventory
4. Credentials: Device machine credential
5. Extra Variables:
   ```yaml
   scp_server: 10.0.0.100
   scp_username: admin
   scp_password: SecurePass123
   new_ios_version: 17.9.4a
   new_image_name: c9300-universalk9.17.09.04a.SPA.bin
   new_image_md5: a1b2c3d4e5f6...
   ```

### 2. Upgrade Execution (`02_upgrade.yaml`)

**Purpose:** Backs up configuration and prepares for reload

**Tasks:**
- Creates timestamped configuration backup
- Transfers backup to SCP server
- Sets boot variable to new image
- Saves configuration

**AWX Setup:**
1. Create Job Template: `Cisco IOS Upgrade Execution`
2. Playbook: `02_upgrade.yaml`
3. Inventory: Select your Cisco devices inventory
4. Credentials: Device machine credential
5. Extra Variables:
   ```yaml
   scp_server: 10.0.0.100
   scp_username: admin
   scp_password: SecurePass123
   new_image_name: c9300-universalk9.17.09.04a.SPA.bin
   ```

### 3. Post-Upgrade Verification (`03_upgrade_post_check.yaml`)

**Purpose:** Validates successful upgrade

**Tasks:**
- (To be implemented)
- Verify new IOS version
- Check system stability
- Validate interface status
- Confirm routing protocols

### 4. Device Information (`playbook.yaml`)

**Purpose:** Gather comprehensive device information

**Tasks:**
- Collects device facts
- Displays hostname, version, model, serial
- Shows running configuration
- Lists interface status
- Displays VLAN information

**AWX Setup:**
1. Create Job Template: `Cisco Device Information`
2. Playbook: `playbook.yaml`
3. Inventory: Select your Cisco devices inventory
4. Credentials: Device machine credential

## Complete Upgrade Workflow in AWX

### Initial AWX Setup

1. **Create Project:**
   - Name: `Cisco IOS Upgrade`
   - SCM Type: Git (or Manual if using local files)
   - SCM URL: Your repository URL
   - Project Path: `/var/lib/awx/projects/network_project`

2. **Create Inventory:**
   - Add your Cisco devices as hosts
   - Configure group variables for cisco_devices group

3. **Create Credentials:**
   - Device SSH credential (Machine type)
   - SCP server credential (if needed)

4. **Create Job Templates** (see individual playbook sections above)

### Step-by-Step Execution

1. **Pre-flight checks:**
   - Launch Job Template: `Cisco Device Information`
   - Review device facts and current state

2. **Run pre-upgrade checks:**
   - Launch Job Template: `Cisco IOS Pre-Upgrade Check`
   - Provide Extra Variables (scp_server, new_ios_version, etc.)
   - Verify image copy and MD5 validation successful

3. **Execute upgrade:**
   - Launch Job Template: `Cisco IOS Upgrade Execution`
   - Configuration backup will be created
   - Boot variable will be set to new image

4. **Manually reload devices:**
   - SSH to each device and execute: `reload in 1`
   - Or create a separate Job Template with reload command

5. **Verify upgrade (after reload):**
   - Launch Job Template: `Cisco IOS Post-Upgrade Check`
   - Verify new version is running

### Using AWX Surveys

Create a Survey for your Job Template to make variable input easier:

**Survey Questions:**
- SCP Server IP
- SCP Username
- SCP Password (password type)
- New IOS Version
- New Image Name
- MD5 Checksum

This allows operators to launch jobs without manually editing Extra Variables.

## Safety Features

- **Version Check:** Prevents upgrade if current version matches target
- **Memory Validation:** Ensures sufficient flash space (2GB minimum)
- **Image Verification:** MD5 checksum validation
- **Configuration Backup:** Automatic backup before changes
- **Duplicate Prevention:** Skips image copy if already present
- **Register Validation:** Ensures proper boot configuration

## Troubleshooting

### Common Issues

**Issue:** Insufficient memory error
```
Solution: Free up flash space by deleting old images
```

**Issue:** SCP transfer fails
```
Solution: Verify SCP server connectivity and credentials
         Check firewall rules
```

**Issue:** MD5 verification fails
```
Solution: Verify MD5 hash is correct
         Re-download image from Cisco
         Check for corruption during transfer
```

**Issue:** Device not accessible
```
Solution: Verify SSH connectivity
         Check ansible_host, ansible_user, ansible_password
         Ensure enable password is configured if required
```

## Best Practices

1. **Test in lab environment first**
2. **Schedule maintenance windows**
3. **Verify backups before upgrade**
4. **Upgrade one device at a time in production**
5. **Keep rollback images on device**
6. **Document device configurations**
7. **Have console access available**
8. **Monitor device during upgrade**

## Security Considerations

- **Store credentials in AWX:** Use AWX Credentials feature for sensitive data
- **Use RBAC:** Configure role-based access control in AWX
- **Restrict SCP server access:** Limit network access to SCP server
- **Enable job auditing:** Review AWX job history and logs
- **Secure AWX instance:** Keep AWX updated and properly secured
- **Use SSH keys:** Configure SSH key-based authentication when possible

### AWX Credentials Management

1. **Create Custom Credential Type** for SCP variables:
   - Input Configuration:
     ```yaml
     fields:
       - id: scp_server
         type: string
         label: SCP Server
       - id: scp_username
         type: string
         label: SCP Username
       - id: scp_password
         type: string
         label: SCP Password
         secret: true
     ```
   - Injector Configuration:
     ```yaml
     extra_vars:
       scp_server: '{{ scp_server }}'
       scp_username: '{{ scp_username }}'
       scp_password: '{{ scp_password }}'
     ```

2. **Attach credential to Job Template** to avoid exposing passwords in Extra Variables

## Support & Compatibility

**Tested On:**
- Cisco Catalyst 9300 Series
- Cisco ISR 4000 Series
- IOS-XE 16.x and 17.x

**AWX Version:** 19.0+

**Ansible Version:** 2.9+

**Collection Version:** cisco.ios 4.0+

## License

This project is provided as-is for network automation purposes.

## Contributing

Feel free to submit issues, fork the repository, and create pull requests for any improvements.

## Author

Network Automation Team

---

**Last Updated:** December 2025
