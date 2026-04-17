---
title: "Set up EntraID for authentication and user provisioning with Posit Workbench"
author: "Lisa Anders"
date: last-modified
format:
  html:
    toc: true
    toc-depth: 3
    code-copy: true
    code-overflow: wrap
---

## Overview

This guide provides a step-by-step approach to configure Microsoft EntraID for user provisioning and as an authentication provider for Workbench using OpenID Connect (OIDC). Where possible, Posit suggests using OIDC with Workbench to enable the most features (e.g., integrating with Azure and AWS Managed Credentials).

## What you will accomplish

By the end of this guide, your Workbench server will:

- Authenticate users against EntraID using OIDC
- Automatically provision user accounts and home directories using SCIM or JIT provisioning

## Prerequisites

Before beginning this configuration, you must have:

- Workbench installed and accessible
- Administrative access to EntraID with the ability to create and modify applications
- Administrative command line access to the Workbench server
- Sudo privileges on the Workbench server
- Valid SSL certificate configurations with verified access over HTTPS for Workbench
  - If you use an external proxy or load balancer, you must configure the front-door address with an SSL certificate.

If you are using SCIM for user provisioning, you must have:

- A secure connection for Azure. The certificate must be a Certificate Authority (CA) signed certificate (i.e., it cannot be self-signed)
- Networking access between the Posit servers and the Entra ID service with port 443 (for HTTPS) open

If configuring with Slurm, additional networking considerations apply. See the [Slurm integration documentation](https://docs.posit.co/ide/server-pro/admin/integration/launcher-slurm.html) for details.

## Choose your user provisioning strategy

This guide covers two provisioning approaches. Choose the one that fits your environment.

### JIT provisioning

Just in time (JIT) provisioning creates user accounts on-demand, removing the need for pre-provisioning users. It also reduces the upfront setup and ongoing maintenance typically associated with a full SCIM integration or traditional directory syncs like LDAP, SSSD, or Active Directory. JIT is simpler to configure but provides limited user lifecycle management. Use JIT if:

- You cannot configure HTTPS with a CA-signed certificate
- EntraID does not have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2` (e.g., often the case for air-gapped Workbench deployments)
- You do not need to manage user deactivation through EntraID

If you need JIT provisioning, follow the [JIT provisioning](#configure-jit) section after completing authentication configuration.

### SCIM provisioning

SCIM provisioning is a good option to consider for its ability to to create and manage assignemnt of users to group if that is needed for downstream workflows. You can use SCIM if your environment meets these requirements:

Generally, we recommend System for Cross-domain Identity Management (SCIM) provisioning. You can use SCIM if your environment meets these requirements:

- You have configured Workbench with HTTPS using a Certificate Authority (CA) signed certificate
- EntraID must have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2`
- You want to manage the full user lifecycle (creation, updates, deactivation) through EntraID
- You want centralized user and group management

If you meet these requirements, follow the [SCIM provisioning](#configure-scim) section after completing authentication configuration.

## Configure authentication

Both provisioning strategies require OIDC authentication. Complete these steps before proceeding to user provisioning configuration.

In addition to access to the hosted web domain, Workbench users need to be provisioned as linux users with a home directory.

### Step 1: Create the user authentication application in Microsoft Entra ID {#create-entra-application}

Create one application per product that needs authentication.

When you create an app registration, EntraID automatically creates an Enterprise application. Both of these together will allow set up of authentication and managing user access.

1. In the Entra ID portal, navigate to **App registrations > New registration**.
2. Enter a name (for example, "Posit-Workbench-Production").
3. Set the **Redirect URI** type to **Web** and enter: `https://<workbench-hostname>/openid/callback`.
4. Select **Register**.
5. Navigate to **Overview > Endpoints** and copy the **OpenID Connect metadata document** URL.
6. In the **Overview** page, copy the **Application (client) ID**. You will need this later.
7. Navigate to **Manage > Certificates & secrets** and create a new **Client secret**.
8. Copy the **Value** of the secret immediately. You will need this in the next step.
9. Under **Manage > Properties**, ensure **Assignment required?** is set to **Yes**, then assign the appropriate users or groups under **Users and groups**.

::: {.callout-note}
EntraID has a notion of app registration and enterprise application. The app registration is used to register external access for an application. The enterprise application is tenant specific and is what controls authentication and (optionally) user provisioning. The order of creation does not matter but them being associated with each other does. This guide shows starting with the application registration (the enterprise application is auto created) since there are already pre-developed templates that can be used. The issue that can be encountered is that Microsoft tries to hide menus to be helpful.

If the provisioning pane is not visible from the enterprise application, delete the app registration and re-start the creation, beginning with the enterprise application and the option to "integrate with an un-listed app".
:::



### Step 2: Encrypt secrets {#encrypt-secrets}

1. On the Workbench server, encrypt your client secret following the [documented encryption steps in the Workbench Admin Guide](https://docs.posit.co/ide/server-pro/admin/hardening/encryption.html#step-re-encrypt-configuration-values):

```{.bash filename="Terminal"}
sudo rstudio-server encrypt-password
```

### Step 3: Configure Workbench for user authentication with OIDC {#configure-workbench-oidc}

1. On the Workbench server, create the file `/etc/rstudio/openid-client-secret` that will include the parameters for the encrypted client secret and the client-id.

2. Add the following content, replacing the placeholder values with your Client ID and Client secret from Step 1:

```{.bash filename="/etc/rstudio/openid-client-secret"}
client-id=<your-client-id>
client-secret=<your-client-secret>
```

The `client id` is the `Application (client) ID` from EntraID. The `client secret` is the encrypted value from above that came from the `Client Secret Value` field in EntraID.

3. Set proper permissions on the `openid-client-secret` file:

```{.bash filename="Terminal"}
sudo chmod 0600 /etc/rstudio/openid-client-secret
sudo chown rstudio-server:rstudio-server /etc/rstudio/openid-client-secret
```

4. Open the Workbench configuration file `/etc/rstudio/rserver.conf` to edit it to use openid.

5. Add the following lines, replacing `<issuer-url>` with the issuer value from Step 5:

```{.bash filename="/etc/rstudio/rserver.conf"}
# Enable OpenID Connect authentication
auth-openid=1

# Configure Entra ID as the OpenID provider
auth-openid-issuer=<issuer-url>

# Configure username claim (usually email or UPN)
auth-openid-username-claim=preferred_username
```

The `auth-openid-issuer` value will be of the form "https://login.microsoftonline.com/{tenant-id}/v2.0".

8. Restart Workbench services:

```{.bash filename="Terminal"}
sudo systemctl restart rstudio-server
sudo systemctl restart rstudio-launcher
```

9. Verify that Workbench has restarted successfully:

```{.bash filename="Terminal"}
sudo systemctl status rstudio-server
```

At this point, you have configured authentication. Users assigned to the Workbench application in EntraID can authenticate, but they cannot start sessions until you configure user provisioning.

## User provisioning

Workbench requires users to have local or networked system accounts. These need to be linux users complete with home directories.

You must set up local system accounts by using `useradd` or network services such as LDAP or Active Directory, JIT or SCIM, and then map authenticating users to these accounts.

### Manual setup (recommended for trials)

If you do not have many users and do not expect many changes in usership, individual provisioning might be the easiest choice. Similarly, if there are issues with the other provisioning methods, this is always an option.

The username must match the preferred_username claim from Entra ID. Alternatively, you can configure Workbench to use a different claim with the `auth-openid-username-claim=` parameter in `/etc/rstudio/rserver.conf`.

```{.bash filename="Terminal"}
# Provision users
sudo useradd myusername -m

# Add users to whichever group is being used to control Workbench access
sudo groupadd myworkbenchusersgroup
```

### Step 1: Configure Workbench for user provisioning {#configure-user-provisioning}

#### JIT

1. Open the Workbench configuration file `/etc/rstudio/rserver.conf` to edit it for JIT. 

    
2. Add the following lines:

```{.bash filename="/etc/rstudio/rserver.conf"}
# Enable user provisioning
user-provisioning-enabled=1

# Enable user provisioning with JIT registration instead of SCIM
user-provisioning-register-on-first-login=1

# Enable Pluggable Authentication Modules (PAM) sessions for home directory creation
auth-pam-sessions-enabled=1

# Optional: Configure starting UID (default is 1000)
#user-provisioning-start-uid=1000

# Optional: Set custom home directory path (default is /home)
#user-homedir-path=/mnt/home
```

#### SCIM

1. Open the Workbench configuration file `/etc/rstudio/rserver.conf` to edit it for SCIM.

2. Add the following lines:

    ```{.bash filename="/etc/rstudio/rserver.conf"}
    # Enable user provisioning
    user-provisioning-enabled=1

    # Enable Pluggable Authentication Modules (PAM) sessions for home directory creation
    auth-pam-sessions-enabled=1
    ```

### Step 2: Configure home directory creation {#configure-home-directory}

Workbench relies on PAM to create user home directories. Configure your operating system accordingly.

::: {.panel-tabset}

#### Ubuntu and Debian

```{.bash filename="Terminal"}
# Install the PAM mkhomedir module if not already installed
sudo apt update
sudo apt install -y libpam-modules

# Configure automatic home directory creation
echo "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022" | sudo tee -a /etc/pam.d/common-session
```

#### RHEL and Rocky

```{.bash filename="Terminal"}
# Install required packages
sudo dnf install -y oddjob-mkhomedir authselect

# Enable the oddjobd service for home directory creation
sudo systemctl enable --now oddjobd.service

# Enable the home directory creation feature
authselect enable-feature with-mkhomedir
authselect apply-changes
```

:::

### Step 3: Configure NSS module {#configure-nss}

Workbench includes a Name Service Switch (NSS) module that allows the operating system to resolve usernames based on the Workbench user service.

::: {.panel-tabset}

#### Ubuntu and Debian

1. Verify the NSS module is installed:

```{.bash filename="Terminal"}
ls -l /usr/lib/x86_64-linux-gnu/libnss_pwb.so.2
```

2. Modify the`/etc/nsswitch.conf` NSS configuration file lines `passwd`, `group`, and `shadow` to include `pwb` if it is not already included:

```{.bash filename="/etc/nsswitch.conf"}
passwd:         files systemd pwb
group:          files systemd pwb
shadow:         files pwb
```

:::{.callout-note}
If you have SSSD or Active Directory configured, ensure `pwb` appears before `sssd` in the configuration to prioritize Workbench users.
:::

#### RHEL and Rocky

1. Verify the NSS module is installed:
    
    ```{.bash filename="Terminal"}
    ls -l /usr/lib64/libnss_pwb.so.2
    ```

2. Create a custom authselect profile:

```{.bash filename="Terminal"}
sudo authselect create-profile pwb --base-on=minimal
```

:::{.callout-note}
 If you have SSSD configured, use `--base-on=sssd` instead of `--base-on=minimal`.
:::

3. Modify the `passwd`, `group`, and `shadow` lines to include `pwb` in the profile configuration `/etc/authselect/custom/pwb/nsswitch.conf`:

```{.bash filename="/etc/authselect/custom/pwb/nsswitch.conf"}
passwd:     files {if "with-altfiles":altfiles }systemd pwb {exclude if "with-custom-passwd"}
group:      files {if "with-altfiles":altfiles }systemd pwb {exclude if "with-custom-group"}
shadow:     files pwb                                       {exclude if "with-custom-shadow"}
```

:::{.callout-note}
If you have SSSD or Active Directory configured, ensure `pwb` appears before `sssd` in the configuration to prioritize Workbench users.
:::

5. Activate the profile:

```{.bash filename="Terminal"}
sudo authselect select custom/pwb with-mkhomedir
sudo authselect apply-changes
```

### Step 4: Disable NSCD caching {#disable-nscd}

Name Service Cache Daemon (NSCD) caching can interfere with the Workbench user provisioning process. Disable it for user and group lookups. Workbench user provisioning implements its own caching mechanism, preventing impact to performance. 

First, check if you are using nscd:

```{.bash filename="Terminal"}
sudo systemctl status nscd
```

If nscd is installed and running, make the following changes.

1. Add or modify these lines:

    ```{.ini filename="/etc/nscd.conf"}
    enable-cache passwd no
    enable-cache group no
    ```

2. Restart NSCD:

    ```{.bash filename="Terminal"}
    sudo systemctl restart nscd
    ```

### Step 6: Generate SCIM authentication token {#generate-scim-token}

:::{.callout-note}
Only proceed with this step if you are using SCIM for user provisioning. Skip this step if you are using JIT.
:::

1. Generate a SCIM token:

```{.bash filename="Terminal"}
sudo rstudio-server user-service generate-token "EntraID SCIM Token"
```

### Step 7: Configure the EntraID SCIM provisioning {#configure-entraid-scim}

:::{.callout-note}
Only proceed with this step if you are using SCIM for user provisioning. Skip this step if you are using JIT.
:::

1. Open the enterprise Entra ID application created previously

2. Under **Manage** select **Provisioning**, set **Provisioning Mode** to **Automatic**.

3. In **Admin Credentials**:

    - **Tenant URL**: `https://<workbench-hostname>/scim/v2`

    - **Secret Token**: Paste the token from Step 1.

4. Under **Provisioning** click on **Mappings**and click on **Provision Microsoft Entra ID Users**, review the list and make sure the mappings and attributes are correct. 

5. Test the connection and save.

### Step 8: (Optional) Configure EntraID SCIM group provisioning {#configure-scim-groups}

:::{.callout-note}
Only proceed with this step if you are using SCIM for user provisioning. Skip this step if you are using JIT.
:::

EntraID can synchronize groups assigned to the Workbench application. This is optional but recommended if you use groups for access control. Groups must be assigned to the application to be provisioned. 

1. Open the enterprise Entra ID application created previously
2. Under **Manage** select **Provisioning** and set the **Provisioning Status** to **On** and save
3. Go to **Manage** select **Users and groups** and select the user and groups that should be added. 

:::{.callout-note}
EntraID limits group membership to 150. If a user is a member of more than 150 groups, then their group list is concatenated, potentially missing important ones that are needed inside Posit Connect or Workbench.
:::

### Step 9: Restart Workbench {#restart-workbench}

Restart Workbench and check the status and logs for any issues.

```{.bash filename="Terminal"}
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

EntraID now automatically provisions users assigned to the Workbench application. Proceed to the [Verification](#verification) section to test your configuration.

## Verification

After completing configuration, test that authentication and provisioning work correctly and that Workbench is able to start successfully without error messages.

### For JIT provisioning

1. In EntraID, assign a test user to the Workbench application.

2. Have the test user navigate to the Workbench URL and sign in using their EntraID credentials.

3. After the user’s first successful login, verify that Workbench created the account:

    ```{.bash filename="Terminal"}
    sudo rstudio-server list-users
    ```

    The test user should appear in the output.

4. Verify that the user can start a session successfully.

### For SCIM provisioning

1. In EntraID, assign a test user to both applications: the Workbench OAuth application and the Workbench SAML SCIM application.

2. Verify that Workbench provisioned the user:

    ```{.bash filename="Terminal"}
    sudo rstudio-server list-users
    ```

    The test user should appear in the output.

3. Have the test user navigate to the Workbench URL and sign in using their EntraID credentials.

4. Verify that the user can start a session successfully.


## Troubleshooting

Use the [pamtester](https://docs.posit.co/ide/server-pro/admin/access_and_security/pam_sessions.html#testing-and-troubleshooting) utility for additional testing:

```{.bash filename="Terminal"}
/usr/lib/rstudio-server/bin/pamtester --verbose rstudio <user> authenticate acct_mgmt setcred open_session close_session
```

In case of issues, you can reach out to our [support team](https://docs.posit.co/support.html). Please make sure to include a copy of your diagnostic as well as all error messages and screenshots of your configured applications in EntraID.

### Complete minimal config example

```{.bash filename="/etc/rstudio/rserver.conf"}
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

## Related documentation

- [Posit Workbench OpenID Connect Authentication](https://docs.posit.co/ide/server-pro/admin/authenticating_users/openid_connect_authentication.html)
- [Enabling OpenID Connect](https://docs.posit.co/ide/server-pro/admin/authenticating_users/openid_connect_authentication.html#enabling-openid-connect)
- [Configuring Microsoft Entra ID for SAML in Posit Workbench](https://docs.posit.co/ide/server-pro/admin/authenticating_users/integrated_providers/azure_ad_saml.html)
- [Microsoft Entra ID with OpenID Connect (Posit Connect)](https://docs.posit.co/connect/admin/authentication/oauth2-openid-based/entra-id-openid-connect/#configuration)
- [Posit Workbench User Provisioning](https://docs.posit.co/ide/server-pro/admin/user_provisioning/user_provisioning.html)
- [Posit Workbench Just-in-Time Provisioning](https://docs.posit.co/ide/server-pro/admin/user_provisioning/just_in_time_provisioning.html)

<!-- internal resources
Azure OIDC Custom Claim - Edited Configuration: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1487011966/Azure+OIDC+Custom+Claim+-+Edited+Configuration>

Azure OIDC Custom Claim Configuration: https://positpbc.atlassian.net/wiki/spaces/SE/pages/1655963671/Azure+OIDC+Custom+Claim+Configuration

Azure OIDC Custom Claims - Configuration with Workbench and Connect examples: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/1487011966/Azure+OIDC+Custom+Claims+-+Configuration+with+Workbench+and+Connect+examples

2023-10-31 Setting up Workbench with Microsoft Entra ID using OIDC: <https://positpbc.atlassian.net/wiki/spaces/SE/pages/491389027/2023-10-31+Setting+up+Workbench+with+Microsoft+Entra+ID+using+OIDC>

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
-->
