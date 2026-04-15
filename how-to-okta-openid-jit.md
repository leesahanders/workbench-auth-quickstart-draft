---
title: "Set up Okta for authentication and user provisioning with Posit Workbench"

author: "Sam Edwardes"
date: last-modified
format:
  html:
    toc: true
    toc-depth: 3
    code-copy: true
    code-overflow: wrap
---

## Overview

This guide provides a step-by-step approach to configure Okta for user provisioning and as an authentication provider for Workbench using OpenID Connect (OIDC). Where possible, Posit suggests using OIDC with Workbench to enable the most features (e.g., integrating with Azure and AWS Managed Credentials).

## What you will accomplish

By the end of this guide, your Workbench server will:

- Authenticate users against Okta using OIDC
- Automatically provision user accounts and home directories using SCIM or JIT provisioning

## Prerequisites

Before beginning this configuration, you must have:

- Workbench installed and accessible
- Administrative access to your Okta organization
- Administrative command line access to the Workbench server
- Sudo privileges on the Workbench server
- Valid SSL certificate configurations with verified access over HTTPS for Workbench.
  - If you use an external proxy or load balancer, you must configure the front-door address with an SSL certificate.

## Choose your user provisioning strategy

This guide covers two provisioning approaches. Choose the one that fits your environment.

### Use System for Cross-domain Identity Management (SCIM) provisioning if you can meet all requirements

SCIM is the recommended approach. You can use SCIM if your environment meets these requirements:

- You have configured Workbench with HTTPS using a Certificate Authority (CA) signed certificate
- Okta must have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2`
- You want to manage the full user lifecycle (creation, updates, deactivation) through Okta
- You want centralized user and group management

If you meet these requirements, follow the [SCIM provisioning](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html#scim-provisioning) section after completing authentication configuration.

### Use Just in Time (JIT) provisioning

JIT provisioning creates user accounts on-demand, removing the need for pre-provisioning users. It also reduces the upfront setup and ongoing maintenance typically associated with a full SCIM integration or traditional directory syncs like LDAP, SSSD, or Active Directory. JIT is simpler to configure but provides limited user lifecycle management. Use JIT if:

- You cannot configure HTTPS with a CA-signed certificate
- Okta does not have connectivity to the Workbench SCIM API endpoints at `https://<workbench-hostname>/scim/v2` (e.g., often the case for air-gapped Workbench deployments)
- You do not need to manage user deactivation through Okta

If you need JIT provisioning, follow the [JIT provisioning](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html#jit-provisioning) section after completing authentication configuration.

## Configure authentication

Both provisioning strategies require OpenID Connect authentication. Complete these steps before proceeding to provisioning configuration.

### Step 1: Create the Okta application {#create-okta-application}

1. In Okta, navigate to **Applications > Browse App Catalog**.
2. Search for “Posit Workbench”.
3. Select **Add Integration** to add Workbench to your organization.
4. Assign the users or groups that should have access to Workbench.
5. Navigate to the **Posit Workbench App > General** tab.
6. In the **App Settings** section, select **Edit**.
7. Enter the base URL of your Workbench server in the **Workbench Base URL** box. Do not include `https://` (e.g., `workbench.example.com`).
8. Select **Save**.
9. Navigate to the **Posit Workbench App > Sign On** tab.
10. Under **Sign on methods**:
    1. Select **OpenID Connect**.
    2. **Application username format**: `Email prefix`
    3. Select **Done**.
11. Copy the **Client ID** and **Client secret** from the **Sign On** tab. You will need these values in the next step.
12. Copy the URL for **OpenID Provider Metadata**. You will need this value in the next step.

### Step 2: Configure Workbench for OpenID Connect {#configure-workbench-oidc}

1. On the Workbench server, create the file `/etc/rstudio/openid-client-secret`:

    ```{.bash filename="Terminal"}
    sudo nano /etc/rstudio/openid-client-secret
    ```

2. Add the following content, replacing the placeholder values with your Client ID and Client secret from Step 1:

    ```{.bash filename="/etc/rstudio/openid-client-secret"}
    client-id=<your-client-id>
    client-secret=<your-client-secret>
    ```

3. Set proper permissions on the file:

    ```{.bash filename="Terminal"}
    sudo chmod 0600 /etc/rstudio/openid-client-secret
    sudo chown rstudio-server:rstudio-server /etc/rstudio/openid-client-secret
    ```

4. In a web browser, open the metadata URL you copied in Step 1. The URL returns a JSON document containing the Okta OpenID configuration.

5. In the JSON document, locate the `issuer` value. Copy this value.

6. Edit the Workbench configuration file:

    ```{.bash filename="Terminal"}
    sudo nano /etc/rstudio/rserver.conf
    ```

7. Add the following lines, replacing `<issuer-url>` with the issuer value from Step 5:

    ```{.bash filename="/etc/rstudio/rserver.conf"}
    # Enable OpenID Connect authentication
    auth-openid=1

    # Configure Okta as the OpenID provider
    auth-openid-issuer=<issuer-url>

    # Configure username claim
    auth-openid-username-claim=preferred_username
    ```

8. Restart Workbench services:

    ```{.bash filename="Terminal"}
    sudo systemctl restart rstudio-server
    sudo systemctl restart rstudio-launcher
    ```

9. Verify that Workbench has restarted successfully:

    ```{.bash filename="Terminal"}
    sudo systemctl status rstudio-server
    ```


At this point, you have configured authentication. Users assigned to the Workbench application in Okta can authenticate, but they cannot start sessions until you configure user provisioning.

## User provisioning

### Step 1: Configure Workbench for user provisioning {#configure-user-provisioning}

- [SCIM](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)
- [JIT](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)

1. Edit the Workbench configuration file:

    ```{.bash filename="Terminal"}
    sudo nano /etc/rstudio/rserver.conf
    ```

2. Add the following lines:

    ```{.bash filename="/etc/rstudio/rserver.conf"}
    # Enable user provisioning
    user-provisioning-enabled=1

    # Enable Pluggable Authentication Modules (PAM) sessions for home directory creation
    auth-pam-sessions-enabled=1
    ```


### Step 2: Configure home directory creation {#configure-home-directory}

Workbench relies on PAM to create user home directories. Configure your operating system accordingly.

- [Ubuntu / Debian](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)
- [RHEL](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)

```{.bash filename="Terminal"}
# Install the PAM mkhomedir module if not already installed
sudo apt update
sudo apt install -y libpam-modules

# Configure automatic home directory creation
echo -e "\n# Posit Workbench: automatic home directory creation\nsession required pam_mkhomedir.so skel=/etc/skel/ umask=0077" | sudo tee -a /etc/pam.d/common-session
```

### Step 3: Configure NSS module {#configure-nss}

Workbench includes a Name Service Switch (NSS) module that allows the operating system to resolve usernames based on Workbench’s user service.

- [Ubuntu / Debian](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)
- [RHEL](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html)

1. Verify the NSS module is installed:

    ```{.bash filename="Terminal"}
    ls -l /usr/lib/x86_64-linux-gnu/libnss_pwb.so.2
    ```

2. Edit the NSS configuration:

    ```{.bash filename="Terminal"}
    sudo nano /etc/nsswitch.conf
    ```

3. Modify the `passwd`, `group`, and `shadow` lines to include `pwb` if it is not already included:

    ```{.bash filename="/etc/nsswitch.conf"}
    passwd:         files systemd pwb
    group:          files systemd pwb
    shadow:         files pwb
    ```

    :::{.callout-note}
    If you have SSSD or Active Directory configured, ensure `pwb` appears before `sssd` in the configuration to prioritize Workbench users.
    :::


### Step 4: Disable NSCD caching {#disable-nscd}

Name Service Cache Daemon (NSCD) caching can interfere with the Workbench user provisioning process. Disable it for user and group lookups.

First, check if you are using nscd:

```{.bash filename="Terminal"}
sudo systemctl status nscd
```

If nscd is installed and running, make the following changes.

1. Edit the NSCD configuration:

    ```{.bash filename="Terminal"}
    sudo nano /etc/nscd.conf
    ```

2. Add or modify these lines:

    ```{.ini filename="/etc/nscd.conf"}
    enable-cache passwd no
    enable-cache group no
    ```

3. Restart NSCD:

    ```{.bash filename="Terminal"}
    sudo systemctl restart nscd
    ```


### Step 5: Restart Workbench {#restart-workbench}

```{.bash filename="Terminal"}
sudo systemctl stop rstudio-server
sudo systemctl stop rstudio-launcher
sudo systemctl start rstudio-launcher
sudo systemctl start rstudio-server
```

Verify that Workbench has restarted successfully:

```{.bash filename="Terminal"}
sudo systemctl status rstudio-server
sudo systemctl status rstudio-launcher
```

### Step 6: Generate SCIM authentication token {#generate-scim-token}

Workbench uses a bearer token to authenticate SCIM requests from Okta.

1. Generate a token:

    ```{.bash filename="Terminal"}
    sudo rstudio-server user-service generate-token "Okta User Provisioning Token"
    ```

2. Copy the token value. You will need this in the next step.

    :::{.callout-important}
    Store this token securely. Anyone with this token can create, modify, or delete users in Workbench.
    :::


### Step 7: Configure Okta SCIM provisioning {#configure-okta-scim}

:::{.callout-note}
Only proceed with this step if you are using SCIM for user provisioning. Skip this step if you are using JIT.
:::

The Okta Marketplace integration for [Workbench](https://www.okta.com/integrations/posit-workbench/) does not support user provisioning with SCIM. This limitation means that you must create a separate app integration for user provisioning. This second app integration for user provisioning must use SAML because Okta custom OIDC applications do not support user provisioning (see Okta support [article](https://support.okta.com/help/s/article/configure-scim-for-a-custom-oidc-app?language=en_US) for more details).

1. In Okta, navigate to **Applications > Create App Integration**.
2. Select **SAML 2.0**.
3. Use the name “Posit Workbench SCIM” (you can use any name you like, but we suggest making it clear that this app integration is for SCIM).
4. Fill out the following **SAML Settings**:
    1. **Single sign-on URL**: `https://<workbench-hostname>/saml/acs` (include `https://`)
    2. **Audience URI (SP Entity ID)**: `https://<workbench-hostname>/saml/metadata` (include `https://`)
    3. **Name ID format**: `Unspecified`
    4. **Application username**: `Email prefix`
5. Select the **General** tab.
6. Select **Edit** for the **App Settings** section:
    1. **Provisioning**: check **SCIM**
7. Select the **Provisioning** tab.
8. Select **Integration**.
9. Select **Edit** in the **SCIM Connection** section.
10. Configure the following settings:
    - **SCIM connector base URL**: Enter `https://<workbench-hostname>/scim/v2`
    - **Unique identifier field for users**: Enter `userName`
    - **Supported provisioning actions**: Select
        - **Push New Users**
        - **Push Profile Updates**
        - **Push groups**
    - **Authentication Mode**: Select **HTTP Header**. In the **Authorization** box, paste the token you generated in Step 6. Select **Test Connector Configuration** to verify the connection.
11. If the test succeeds, select **Save**.
12. Select the **To App** tab.
13. Select **Edit**.
14. Enable the following options:
    - **Create Users**
    - **Update User Attributes**
    - **Deactivate Users**
15. Select **Save**.

### Step 8: (Optional) Configure Okta SCIM group provisioning {#configure-scim-groups}
Okta can synchronize groups assigned to the Workbench application. This is optional but recommended if you use groups for access control.

1. Select the **Push Groups** tab.
2. Select **+ Push Groups**.
3. Find the group you want to push and select **Save**.
4. If successful, the **Push Status** changes to **Active**.

Okta now automatically provisions users assigned to the Workbench application in Workbench. Proceed to the [Verification](https://connect.posit.it/content/055ff32f-42e1-4652-b651-8e5d01b4f2d1/vQKa4NXpt/pwb-okta-guide.html#verification) section to test your configuration.

## Verification

After completing configuration, test that authentication and provisioning work correctly and that Workbench is able to start successfully without error messages.

### For SCIM provisioning

1. In Okta, assign a test user to both applications: the Workbench OAuth application and the Workbench SAML SCIM application.

2. Verify that Workbench provisioned the user:

    ```{.bash filename="Terminal"}
    sudo rstudio-server list-users
    ```

    The test user should appear in the output.

3. Have the test user navigate to the Workbench URL and sign in using their Okta credentials.

4. Verify that the user can start a session successfully.


### For JIT provisioning

1. In Okta, assign a test user to the Workbench application.

2. Have the test user navigate to the Workbench URL and sign in using their Okta credentials.

3. After the user’s first successful login, verify that Workbench created the account:

    ```{.bash filename="Terminal"}
    sudo rstudio-server list-users
    ```

    The test user should appear in the output.

4. Verify that the user can start a session successfully.


## Related documentation

- [Posit Workbench OpenID Connect Authentication](https://docs.posit.co/ide/server-pro/admin/authenticating_users/openid_connect_authentication.html)
- [Posit Workbench User Provisioning](https://docs.posit.co/ide/server-pro/admin/user_provisioning/user_provisioning.html)
- [Posit Workbench Just-in-Time Provisioning](https://docs.posit.co/ide/server-pro/admin/user_provisioning/just_in_time_provisioning.html)
- [Okta OpenID Connect Documentation](https://developer.okta.com/docs/concepts/oauth-openid/)
- [Okta SCIM Documentation](https://developer.okta.com/docs/concepts/scim/)
