---
title: "How to set up EntraID for authentication and user provisioning with Posit Workbench"
author: "Lisa Anders"
date: last-modified
format:
  html:
    toc: true
    toc-depth: 3
    code-copy: true
    code-overflow: wrap
---

# Configure EntraID for authentication and user provisioning with Posit Workbench

Step-by-step guide for configuring EntraID authentication using OpenID Connect with System for Cross-domain Identity Management (SCIM) or Just in Time (JIT) user provisioning.

# Overview 

Microsoft Entra ID (Entra ID) is a common authentication choice among customers for its ability to support Single Sign-On (SSO). This guide will show an opinionated setup and configuration Posit Workbench with Entra ID using OpenID Connect (OIDC). We suggest using OIDC with Posit Workbench as it enables the most features (e.g., Azure and AWS Managed Credentials).

## What you will accomplish

By the end of this guide, your Workbench server will:

- Authenticate users against EntraID using OIDC
- Automatically provision user accounts and home directories using SCIM or JIT provisioning

# Pre requisites 

Before beginning this configuration:

- You must have Posit Workbench installed and accessiblet 
- You must have administrative access to EntraID with the ability to create and modify applications
- You must have administrative command line access to the Workbench server
- You must have sudo privileges on the Workbench server
- Valid SSL certificate configurations with verified access over HTTPS for Workbench
  - If an external proxy or load balancer is used, the front-door address must be configured with an SSL certificate.

If SCIM for user provisioning is needed then the additional requirements will need to be satisfied: 

- Azure requires a secure connection, the certificate must be a Certificate Authority (CA) signed certificate (cannot be self-signed)
- Networking access between the Posit servers and the Entra ID service with port 443 (for HTTPS) open.

If configuring with Slurm then additional networking considerations will need to be taken into account. Slurm configuration is documented [here](https://docs.posit.co/ide/server-pro/admin/integration/launcher-slurm.html). 

## Choose your user provisioning strategy

### Use System for Cross-domain Identity Management (SCIM) provisioning if you can meet all requirements

SCIM is the recommended approach. You can use SCIM if your environment meets these requirements:

- You have configured Workbench with HTTPS using a Certificate Authority (CA) signed certificate
- EntraID must have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2`
- You want to manage the full user lifecycle (creation, updates, deactivation) through Okta
- You want centralized user and group management

If you meet these requirements, follow the [SCIM provisioning TODO]() section after completing authentication configuration.

### Use Just in Time (JIT) provisioning

JIT provisioning offers distinct advantages by (1) creating user accounts on-demand, removing the need for pre-provisioning users, and (2) reducing the upfront setup and ongoing maintenance typically associated with a full SCIM integration or managing traditional directory syncs like LDAP/SSSD/Active Directory. It is is simpler to configure but provides limited user lifecycle management. Use JIT if:

- You cannot configure HTTPS with a CA-signed certificate
- EntraID does not have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2` (e.g., often the case for air-gapped Workbench deployments)
- You do not need to manage user deactivation through Okta

If you need JIT provisioning, follow the [JIT provisioning TODO]() section after completing authentication configuration.

## Posit Workbench Set Up Overview 

In addition to access to the hosted web domain, Workbench users need to be provisioned as linux users with a home directory. 

The steps needed to set up Workbench are: 

- Encrypt secrets
- Configure authentication in the Workbench config file ``
- Configure user provisioning by enabling home directory creation 
- Configure nss switch and the workbench pam auth profile 
- Configure user provisioning in the Workbench config file ``
- Restart Workbench
- Test and verify operation

## Configure authentication

### Create the user authentication application in Microsoft Entra ID

One application will need to be created per product needing authentication. 

When an app registration is created, EntraID will automatically create an Enterprise application. Both of these together will allow set up of authentication and managing user access. 

1. In the Entra ID portal, navigate to **App registrations** → **New registration**.
    
2. Enter a name (e.g., "Posit-Workbench-Production").
    
3. Set the **Redirect URI** type to **Web** and enter: `https://<workbench-hostname>/openid/callback`.
    
4. Select **Register**.
    
5. Navigate to **Overview** → **Endpoints** and copy the **OpenID Connect metadata document** URL.
    
6. In the **Overview** page, copy the **Application (client) ID**. You will need this later.
    
7. Navigate to **Manage** → **Certificates & secrets** and create a new **Client secret**.
    
8. Copy the **Value** of the secret immediately. You will need this in the next step.
    
9. Under **Manage** → **Properties**, ensure **Assignment required?** is set to **Yes**, then assign the appropriate users/groups under **Users and groups**.

> Note: EntraID has a notion of app registration and enterprise application. The app registration is used to register external access for an application. The enterprise application is tenant specific and is what controls authentication and (optionally) user provisioning. The order of creation doesn't matter but them being associated with each other does. This guide shows starting with the application registration (the enterprise application will be auto created) since there are already pre-developed templates that can be used. The issue that can be encountered is that Microsoft tries to hide menu's to be helpful. 
> 
> If the provisioning pane isn't visible from the enterprise application then the app registration should be deleted and the creation should be re-started but starting with the enterprise application and the option to "integrate with an un-listed app".

### Configure Workbench for OpenID Connect

#### Encrypt secrets 
1. On the Workbench server, encrypt your client secret following the steps [documented encryption steps in the Workbench Admin Guide](https://docs.posit.co/ide/server-pro/admin/hardening/encryption.html#step-re-encrypt-configuration-values). :

```bash
sudo rstudio-server encrypt-password
```

#### Configure Posit Workbench for user authentication

Create a file `/etc/rstudio/openid-client-secret` that will include the parameters for the encrypted client secret and the client-id. 

```{.bash filename="/etc/rstudio/openid-client-secret"}
client-id="example-example-example-example-example"
client-secret="example"
```

The `client id` is the `Application (client) ID` from EntraID. The `client secret` is the encrypted value from above that came from the `Client Secret Value` field in EntraID. 

Protect the `openid-client-secret` file to restrict access: 

```bash
sudo chmod 0600 /etc/rstudio/openid-client-secret
sudo chown rstudio-server:rstudio-server /etc/rstudio/openid-client-secret
```

Edit the `/etc/rstudio/rserver.conf` file to use openid. 

```{.bash filename="/etc/rstudio/rserver.conf"}
    # Enable OpenID Connect authentication
    auth-openid=1

    # Configure Entra ID as the OpenID provider
    auth-openid-issuer="https://login.microsoftonline.com/{tenant-id}/v2.0"

    # Configure username claim (usually email or UPN)
    auth-openid-username-claim=preferred_username
```

The `auth-openid-issuer` value will be "https://login.microsoftonline.com/{tenant-id}/v2.0". 


## User Provisioning 

Workbench requires users to have local or networked system accounts. These need to be linux users complete with home directories. 

You must set up local system accounts manually, or using network services such as LDAP or Active Directory, and then map authenticating users to these accounts.

### Configure Workbench for user provisioning

#### Release valve: Manual provisioning 

If you don't have many users and don't expect many changes in usership then manually provisioning might be the easiest choice. Similarly, if there are issues with the other provisioning methods this is always an option. 

The key is that the username must match the prefered_username claim coming from Entra ID (or be a claim coming from Entra ID that Workbench can be configured to use with the `auth-openid-username-claim=` parameter in `/etc/rstudio/rserver.conf`). 

```bash
sudo useradd myusername@mycompany.com -m
```

#### System dependencies 

System dependencies should be installed as the root user. While not always required, completing the install as root will make things go smoother. 

##### Ubuntu / Debian

Follow the steps in the [how to guide for setting up JIT](https://github.com/rstudio/docs.rstudio.com/blob/ead3bd8cc2b0d03a32de04a14b7d3f791447191e/how-to-guides/guides/configure-pwb-jit.md). 

```bash
# Install the PAM mkhomedir module if not already installed
sudo apt update
sudo apt install -y libpam-modules

# Edit the PAM configuration to add a line to automatically create user home directories in common-session
echo "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022" | sudo tee -a /etc/pam.d/common-session
```

##### RHEL / Rocky

Follow the steps in the [how to guide for setting up JIT](https://github.com/rstudio/docs.rstudio.com/blob/ead3bd8cc2b0d03a32de04a14b7d3f791447191e/how-to-guides/guides/configure-pwb-jit.md). 

```bash
# Install required packages
sudo dnf install -y oddjob-mkhomedir authselect

# Enable the oddjobd service for home directory creation
sudo systemctl enable --now oddjobd.service

# Enable the home directory creation feature
authselect enable-feature with-mkhomedir
authselect apply-changes
```

#### Configure the NSS module

Follow the steps in the [how to guide for setting up JIT](https://github.com/rstudio/docs.rstudio.com/blob/ead3bd8cc2b0d03a32de04a14b7d3f791447191e/how-to-guides/guides/configure-pwb-jit.md). 

##### Ubuntu / Debian

Modify the `/etc/nsswitch.conf` file to include pwb before sssd (if present) otherwise last in each row. For example it might look something like this (remember to only add pwb, no other parameters): 

```{.bash filename="/etc/nsswitch.conf"}
# Modify the passwd, group, and shadow lines to include pwb at the end
passwd:         files systemd pwb
group:          files systemd pwb
shadow:         files pwb sssd
```

##### RHEL / Rocky

Create an authselect profile: 

```bash
sudo authselect create-profile pwb --base-on=minimal
```

Modify the `/etc/authselect/custom/pwb/nsswitch.conf` file to include pwb before sssd (if present) otherwise last in each row. For example it might look something like this (remember to only add pwb, no other parameters): 

```{.bash filename="/etc/authselect/custom/pwb/nsswitch.conf"}
# Modify the passwd, group, and shadow lines to include pwb at the end
passwd:         files systemd pwb
group:          files systemd pwb
shadow:         files pwb sssd
```

Once saved update the profile with: 

```bash
# Enable the custom profile with mkhomedir support
sudo authselect select custom/pwb with-mkhomedir
sudo authselect apply-changes
```

#### Disable NSCD caching

Follow the steps in the [how to guide for setting up JIT](https://github.com/rstudio/docs.rstudio.com/blob/ead3bd8cc2b0d03a32de04a14b7d3f791447191e/how-to-guides/guides/configure-pwb-jit.md). 

NSCD Caching can cause issues with Workbench's user provisioning process, and so must be disabled by following the steps below. Disabling these caches should not affect system performance, as Workbench user provisioning implements its own caching mechanism.

If NSCD caching utilities aren't installed then continue to the next step. 

```
# Add or modify the following lines
enable-cache passwd no
enable-cache group no
```

#### Configure JIT Provisioning for Posit Workbench 

Follow the steps in the [how to guide for setting up JIT](https://github.com/rstudio/docs.rstudio.com/blob/ead3bd8cc2b0d03a32de04a14b7d3f791447191e/how-to-guides/guides/configure-pwb-jit.md). 

Update the `/etc/rstudio/rserver.conf` config file to enable user provisioning with JIT: 

```{.bash filename="/etc/rstudio/etc/rstudio/rserver.conf"}
# Add the following lines
user-provisioning-enabled=1
user-provisioning-register-on-first-login=1

# Add the following line to enable PAM sessions
auth-pam-sessions-enabled=1

# Optional: Configure starting UID (default is 1000)
#user-provisioning-start-uid=1000

# Optional: Set custom home directory path (default is /home)
#user-homedir-path=/mnt/home
```

#### (Optional) Configure Entra ID SCIM provisioning

If you are using SCIM instead of JIT:

1. Generate a SCIM token:

```bash
sudo rstudio-server user-service generate-token "EntraID SCIM Token"
```
    
2. Open the Entra ID application created previously
    
3. Under **Provisioning**, set **Provisioning Mode** to **Automatic**.
    
4. In **Admin Credentials**:
    
    - **Tenant URL**: `https://<workbench-hostname>/scim/v2`
        
    - **Secret Token**: Paste the token from Step 1.
        
5. Test the connection and save.

#### Restart Posit Workbench

Restart Workbench and check the status and logs for any issues. 

```bash
# Restart the service 
sudo rstudio-server restart
sudo rstudio-launcher restart

# Check the status
sudo systemctl status rstudio-server
sudo systemctl status rstudio-launcher

# Check the most recent logs
sudo tail -n 50 /var/log/rstudio/rstudio-server/rserver.log | grep error*
sudo tail -n 50 /var/log/rstudio/launcher/rstudio-launcher.log
```

## Verification 

After completing configuration, test that authentication and provisioning work correctly and that Workbench is able to start successfully without error messages. 

## Troubleshooting 

In case of issues you can reach out to our [support team](https://docs.posit.co/support.html).Please make sure to include a copy of your diagnostic as well as all error messages and screenshots of your configured applications in EntraID. 

The [pamtester](https://docs.posit.co/ide/server-pro/admin/access_and_security/pam_sessions.html#testing-and-troubleshooting) utility may be used for additional testing: 

```bash
/usr/lib/rstudio-server/bin/pamtester --verbose rstudio <user> authenticate acct_mgmt setcred open_session close_session
```

### EntraID group issues

EntraID limits group membership to 150. If a user is a member of more than 150 groups than their group list will be concatenated, potentially missing important ones that are needed inside Connect or Workbench. 

### Posit Workbench 

#### Complete minimal config example 

```{.bash filename="/etc/rstudio/etc/rstudio/rserver.conf"}
www-port=8787
server-health-check-enabled=1
# auth-minimum-user-id=100
launcher-sessions-enabled=1
# Use localhost as your launcher address
launcher-address=localhost
launcher-port=5559
# Use the https address of your load balancer, proxy, or server
launcher-sessions-callback-address=https://workbench.example.com
launcher-default-cluster=Local

# OpenID
auth-openid=1
auth-openid-issuer=https://login.microsoftonline.com/example-example-example-example-example/v2.0

# JIT
user-provisioning-enabled=1
user-provisioning-register-on-first-login=1
auth-pam-sessions-enabled=1
```

```{.bash filename="/etc/rstudio/launcher.conf"}
[server]
address=127.0.0.1
port=5559
server-user=rstudio-server
admin-group=rstudio-server
authorization-enabled=1
#enable-debug-logging=1

[cluster]
name=Local
type=Local
```

```{.bash filename="/etc/rstudio/openid-client-secret"}
client-id="example-example-example-example-example"
client-secret="example"
```

# Related / Inspiration 

Azure OIDC Custom Claim - Edited Configuration: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1487011966/Azure+OIDC+Custom+Claim+-+Edited+Configuration>

Azure OIDC Custom Claim Configuration: https://positpbc.atlassian.net/wiki/spaces/SE/pages/1655963671/Azure+OIDC+Custom+Claim+Configuration

Azure OIDC Custom Claims - Configuration with Workbench and Connect examples: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1487011966/Azure+OIDC+Custom+Claims+-+Configuration+with+Workbench+and+Connect+examples

2023-10-31 Setting up Workbench with Microsoft Entra ID using OIDC: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/491389027/2023-10-31+Setting+up+Workbench+with+Microsoft+Entra+ID+using+OIDC>

Configuring Microsoft Entra ID for SAML in Posit Workbench: <https://docs.posit.co/ide/server-pro/admin/authenticating_users/integrated_providers/azure_ad_saml.html>

Enabling OpenID Connect: <https://docs.posit.co/ide/server-pro/admin/authenticating_users/openid_connect_authentication.html#enabling-openid-connect>

Microsoft Entra ID (using OpenID Connect): <https://docs.posit.co/connect/admin/authentication/oauth2-openid-based/entra-id-openid-connect/#configuration>

How-To Guide for JIT Implementation (heavily drawn from): <https://github.com/rstudio/docs.rstudio.com/pull/2457>

Configuring Azure EntraID for SAML Auth in Workbench and Connect: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1070825680/Configuring+Azure+EntraID+for+SAML+Auth+in+Workbench+and+Connect>

Can you use the same oauth app on Workbench and Connect? <https://positpbc.slack.com/archives/G02HSFX6BEF/p1753365205864789>

use of nss/pam sessions the system: <https://positpbc.slack.com/archives/C04280LRVQT/p1749492534858039>

Feedback on rough edges in the docs: <https://positpbc.slack.com/archives/CUC92M4TT/p1749498335271989> 

Great call about Connect with a customer that was setting up SAML: (FBL Financial 11/25/2025): <https://us-1237.app.gong.io/call?id=6501570621460575839&email_type=call-ready-notification&xtid=39juvnvgybucyrmq72c>

Configuring LDAP / Active Directory with Posit Team: <https://solutions.posit.co/secure-access/auth/ldap_setup/>

How to JIT by SamC: <https://pub.current.posit.team/public/How-To-JIT/implementing_jit_provisioning.html>

Katie's user provisioning page: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1172177012/Workbench+user+provisioning+set+up+SCIM+JIT> 

Other auth guides: 

- <https://pub.current.posit.team/public/How-To-JIT/implementing_jit_provisioning.html> 
- <https://pub.demo.posit.team/content/b42332f5-06da-49a0-9914-d1bc2c1ef8c9/>
- <https://connect.posit.it/connect/#/apps/055ff32f-42e1-4652-b651-8e5d01b4f2d1/5101>

