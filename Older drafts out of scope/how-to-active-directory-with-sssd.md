---
title: "How to Set Up Active Directory with Posit Workbench"
subtitle: "A Golden Path Guide Using SSSD for Authentication and User Provisioning"
author: "Cam Palmer"
date: last-modified
format:
  html:
    toc: true
    toc-depth: 3
    code-copy: true
    code-overflow: wrap
---

## Overview

This guide provides an opinionated, step-by-step walkthrough for integrating Posit Workbench with Microsoft Active Directory (AD) using SSSD (System Security Services Daemon) for authentication and automatic user provisioning.

The [official Workbench documentation](https://docs.posit.co/ide/server-pro/authenticating_users/active_directory.html) covers Active Directory integration comprehensively. This guide distills that into a single, prescriptive path—making specific choices so you don't have to.

### What you will accomplish

By the end of this guide, your Workbench server will:

-   Authenticate users against your Active Directory domain
-   Automatically provision user accounts and home directories on first login
-   Use Kerberos for secure, password-less authentication to downstream resources

### Prerequisites

Before starting, ensure you have:

| Requirement | Details |
|------------------------------------------|------------------------------|
| Posit Workbench | Installed and accessible, you must have administrative command line access to the Workbench server |
| **Active Directory Domain** | A functioning AD domain you can join |
| **DNS resolution** | Workbench server can resolve AD domain names |
| **Time synchronization** | NTP/Chrony configured (Kerberos requires \<5 min clock skew) |
| **Admin credentials** | AD account with permission to join endpoints to the domain |
| **Firewall rules** | Ports 53, 88, 389, 445, 464, 3268 (TCP/UDP) open to Domain Controller(s) |
| **pamtester** | PAM testing utility (installed in Step 1) |

## Step 1. Install required packages {#install-required-packages}

::: panel-tabset
### RHEL

``` bash
sudo dnf install -y \
  realmd \
  sssd \
  sssd-tools \
  oddjob \
  oddjob-mkhomedir \
  adcli \
  samba-common-tools \
  krb5-workstation \
  pamtester
```

### Ubuntu / Debian

``` bash
sudo apt update
sudo apt install -y \
  realmd \
  sssd \
  sssd-tools \
  libnss-sss \
  libpam-sss \
  adcli \
  samba-common-bin \
  krb5-user \
  packagekit \
  pamtester
```
:::

We use `realmd` because it handles the complexity of domain joining automatically—configuring Kerberos, SSSD, and PAM in one command. This is the recommended approach over manual configuration.

## Step 2. Verify DNS and domain discovery {#verify-dns}

Before joining, confirm your server can discover the AD domain:

``` bash
realm discover YOUR-DOMAIN.COM
```

You should see output similar to:

```         
your-domain.com
  type: kerberos
  realm-name: YOUR-DOMAIN.COM
  domain-name: your-domain.com
  configured: no
  server-software: active-directory
  client-software: sssd
  required-package: sssd-tools
  required-package: sssd
  required-package: adcli
  required-package: samba-common-bin
```

::: callout-important
## If discovery fails

If `realm discover` fails, check:

1.  **DNS**: Can you resolve the domain? `nslookup YOUR-DOMAIN.COM`
2.  **Network**: Can you reach the DC? `nc -zv dc.your-domain.com 389`
3.  **Time**: Is your clock synchronized? `timedatectl status`
:::

## Step 3. Join the domain {#join-domain}

Join your Workbench server to the Active Directory domain:

``` bash
sudo realm join --user=Administrator YOUR-DOMAIN.COM
```

The system will prompt you for the AD administrator password. After successful joining, verify:

``` bash
realm list
```

Expected output:

```         
your-domain.com
  type: kerberos
  realm-name: YOUR-DOMAIN.COM
  domain-name: your-domain.com
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  ...
```

## Step 4. Configure SSSD for Workbench compatibility {#configure-sssd}

The `realm join` command creates a default SSSD configuration that needs adjustments to work properly with Workbench. Edit the SSSD configuration file:

``` bash
sudo nano /etc/sssd/sssd.conf
```

Locate the `[domain/your-domain.com]` section and configure the following settings (add or modify as needed):

``` ini
[domain/your-domain.com]
# ... existing configuration from realm join ...

# CRITICAL: Allow short usernames (required for Workbench)
use_fully_qualified_names = False

# Set home directory path
fallback_homedir = /home/%u

# Enable AD-based access control
access_provider = ad

# CRITICAL: Set GPO to permissive mode (prevents "access denied" after successful auth)
ad_gpo_access_control = permissive

# Optional: Restrict access to specific AD security group
# ad_access_filter = (memberOf=CN=rstudio-users,OU=YourOU,DC=your-domain,DC=com)

# Performance tuning for large directories
enumerate = False
ignore_group_members = True

# Cache credentials for offline authentication
cache_credentials = True

# Extend Kerberos ticket lifetime for long-running jobs
krb5_renewable_lifetime = 7d
krb5_renew_interval = 60m
```

::: callout-important
## Critical SSSD settings explained

**`use_fully_qualified_names = False`**: Allows users to log in with just their username instead of `username@domain.com`. Without this, Workbench cannot match usernames correctly.

**`ad_gpo_access_control = permissive`**: This is the most common cause of "authentication succeeds but login fails" errors. By default, SSSD enforces Active Directory Group Policy Objects (GPO). If the PAM service `rstudio` isn't explicitly allowed by GPO, access is denied even after successful authentication. Setting this to `permissive` bypasses GPO enforcement.

**`ad_access_filter`**: Optional but recommended. Restricts Workbench access to members of a specific AD security group. Replace the example with your actual group DN.
:::

After editing, set proper file permissions (SSSD requires strict permissions):

``` bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
```

Restart SSSD to apply changes:

``` bash
sudo systemctl restart sssd
```

Verify SSSD restarted successfully and check for errors:

``` bash
sudo systemctl status sssd
```

## Step 5. Configure home directory creation {#configure-home-directory}

Workbench relies on PAM to create user home directories on first login.

::: panel-tabset
### RHEL

On RHEL-based systems, use `oddjob-mkhomedir` to ensure proper SELinux contexts:

``` bash
# Enable and start the oddjobd service
sudo systemctl enable --now oddjobd

# Configure PAM to use oddjobd for home directory creation
sudo authselect select sssd with-mkhomedir --force
```

On SELinux-enabled systems (RHEL default), `pam_mkhomedir` creates directories with incorrect security contexts, causing "Permission Denied" errors. `oddjob-mkhomedir` creates directories with proper SELinux labels.

### Ubuntu / Debian

``` bash
# Install the PAM mkhomedir module if not already installed
sudo apt update
sudo apt install -y libpam-modules

# Configure automatic home directory creation
echo -e "\n# Posit Workbench: automatic home directory creation\nsession required pam_mkhomedir.so skel=/etc/skel umask=0077" | sudo tee -a /etc/pam.d/common-session
```
:::

## Step 6. Configure Workbench settings {#configure-workbench}

Edit the Workbench configuration file:

``` bash
sudo nano /etc/rstudio/rserver.conf
```

Add the following line:

``` ini
# Enable Pluggable Authentication Modules (PAM) sessions for home directory creation
auth-pam-sessions-enabled=1
```

This enables PAM sessions, which are required for home directory creation to work properly.

## Step 7. Restart Workbench {#restart-workbench}

Apply the configuration by restarting Workbench:

``` bash
sudo rstudio-server restart
```

Verify that Workbench has restarted successfully:

``` bash
sudo systemctl status rstudio-server
```

## Validation

Test your configuration with these commands:

### 1. Verify user lookup

``` bash
id username
# Should return user information without requiring @domain.com
```

Expected output:

```         
uid=1234567(username) gid=1234567(domain users) groups=...
```

If this fails, verify that `use_fully_qualified_names = False` is set in `/etc/sssd/sssd.conf` and SSSD has been restarted.

### 2. Test Kerberos authentication

``` bash
kinit username@YOUR-DOMAIN.COM
klist
```

### 3. Verify PAM configuration

``` bash
sudo pamtester rstudio username authenticate
```

If `pamtester` succeeds but web login fails, this typically indicates a SSSD access control issue (see Troubleshooting section).

### 4. Test Workbench login

1.  Open your browser to `https://your-workbench-server:8787`
2.  Log in with AD credentials using just the username (not `username@domain.com`)
3.  Verify that the system created a home directory: `ls -la /home/username/`

## Troubleshooting

### Understanding access control layers

SSSD implements multiple layers of access control. **PAM authentication success does not guarantee login success.** Users must pass through all these layers:

1.  **PAM authentication** - Verifies credentials
2.  **SSSD access_provider** - Checks if user has access
3.  **Access filters** - Optional group membership checks (`ad_access_filter`)
4.  **Group Policy Objects (GPO)** - AD policy enforcement

Each layer can independently deny access. Check logs at each layer when troubleshooting.

### Login fails after successful authentication (GPO issues)

**Symptoms**: `pamtester` succeeds, but web login fails with "Permission denied" or "Access denied"

**Log indicators**: Look in `/var/log/sssd/sssd_<domain>.log` for:
```
[ad_gpo_access_send] service rstudio maps to Denied
[ad_gpo_access_done] GPO-based access control failed.
```

**Solution**: Set `ad_gpo_access_control = permissive` in `/etc/sssd/sssd.conf` and restart SSSD:

``` bash
sudo systemctl restart sssd
```

This is the most common cause of "authentication succeeds but login fails" errors with Active Directory integration.

### Username format mismatch

**Symptoms**: User lookup requires full UPN (`username@domain.com`) but Workbench login with short name fails

**Solution**: Ensure `use_fully_qualified_names = False` in `/etc/sssd/sssd.conf`, then restart SSSD:

``` bash
sudo systemctl restart sssd
```

Verify with both:
``` bash
id username
id username@your-domain.com
```

Both should return the same user information.

### User lookup fails (`id` returns "no such user")

Check SSSD configuration and logs:

``` bash
# Clear SSSD cache and restart
sudo sss_cache -E
sudo systemctl restart sssd

# Check SSSD status
sudo systemctl status sssd

# View SSSD logs (increase verbosity if needed)
sudo journalctl -u sssd -f
```

Check domain-specific logs:
``` bash
sudo tail -f /var/log/sssd/sssd_<your-domain>.log
```

### Access denied errors

Check the three key log files:

1.  `/var/log/rstudio/rstudio-server/rserver.log` - Workbench application errors
2.  `/var/log/auth.log` (Ubuntu) or `/var/log/secure` (RHEL) - PAM authentication details
3.  `/var/log/sssd/sssd_<domain>.log` - SSSD access control decisions

Look for error messages containing:
- `pam_sss(rstudio:account): Access denied`
- `GPO-based access control failed`
- `Permission denied`

### SSSD won't start after editing configuration

Check file permissions:

``` bash
# SSSD requires strict permissions
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf

# Restart SSSD
sudo systemctl restart sssd

# Check for configuration errors
sudo systemctl status sssd
```

Look for duplicate configuration entries in `/etc/sssd/sssd.conf`. Each setting should appear only once in the domain section.

### Login is slow (\>30 seconds)

Large AD directories can cause slow lookups. Ensure these settings are in `/etc/sssd/sssd.conf`:

``` ini
enumerate = False
ignore_group_members = True
```

### Clock skew errors

Kerberos requires synchronized time (within 5 minutes). Verify NTP:

``` bash
timedatectl status
# Should show "System clock synchronized: yes"

# If not synchronized:
sudo systemctl enable --now chronyd  # RHEL
sudo systemctl enable --now systemd-timesyncd  # Ubuntu
```

## Summary

You configured Workbench to:

✅ Authenticate users against Active Directory\
✅ Automatically provision user accounts on first login\
✅ Create home directories with proper permissions\
✅ Use Kerberos for secure authentication

### Key choices made in this guide

| Decision | Our Choice | Why |
|-------------------------|------------------------------|-----------------|
| Domain join tool | `realmd` | Automates Kerberos, SSSD, and PAM configuration |
| Authentication daemon | `sssd` | Modern, well-supported, handles caching and offline auth |
| Home directory creation (RHEL) | `oddjob-mkhomedir` | Proper SELinux contexts |
| Home directory creation (Ubuntu) | `pam_mkhomedir` | Standard approach for non-SELinux systems |
| GPO enforcement | `permissive` | Prevents authentication/authorization mismatch issues |