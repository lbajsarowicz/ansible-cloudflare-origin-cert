# Ansible Role: Cloudflare Origin Certificate Management

## Role Goal

This Ansible role automates the management of Cloudflare Origin SSL certificates for your web servers. Its primary functions are to:

1.  **Verify Existence & Expiration**: Check if a valid Cloudflare Origin Certificate already exists for the specified hostnames within a given Cloudflare zone.
2.  **Check Expiration**: If a certificate exists, it verifies if the certificate is due to expire within a configurable period (defaulting to less than 1 year).
3.  **Generate Certificate**: If no certificate exists, or if the existing one is nearing expiration, the role connects to the Cloudflare API to generate a new Origin Certificate.
4.  **Download & Install**: The newly generated certificate and its private key are downloaded and installed to specified paths on the target server.
5.  **CA Root Installation**: Optionally installs the Cloudflare Origin CA root certificate to ensure the full trust chain can be served.
6.  **Web Server Restart**: Notifies a handler to restart your web server to apply the new certificate.

This role aims to ensure your origin server is always equipped with a valid Cloudflare Origin Certificate, minimizing manual intervention and potential downtime due to expired certificates.

## Requirements

* Ansible 2.9 or later (uses `ansible.builtin` modules).
* Access to a Cloudflare account and the target server where the certificate will be installed.
* Python `requests` library on the Ansible control node (often a dependency of the `uri` module for HTTPS).

## Role Variables

The following variables can be configured by the user:

| Variable                       | Required | Default (in handler/role) | Description                                                                                                |
|--------------------------------| -------- |---------------------------| ---------------------------------------------------------------------------------------------------------- |
| `cf_api_token`                 | Yes      | N/A                       | **Sensitive**: Your Cloudflare API Token with permissions to manage SSL certificates for the specified zone. |
| `cf_zone_id`                   | Yes      | N/A                       | The Zone ID of your domain in Cloudflare. See "Obtaining Your Cloudflare Zone ID" below.                     |
| `cf_certificate_hostnames`     | Yes      | N/A                       | A list of hostnames the certificate should cover (e.g., `["example.com", "*.example.com"]`).                |
| `cf_certificate_install_path`  | Yes      | N/A                       | The directory path on the target server where the certificate and key files will be installed.               |
| `cf_certificate_file`          | Yes      | N/A                       | The full path (including filename) for the origin certificate file (e.g., `{{ cf_certificate_install_path }}origin.pem`). |
| `cf_key_file`                  | Yes      | N/A                       | The full path (including filename) for the private key file (e.g., `{{ cf_certificate_install_path }}origin.key`). |
| `cf_certificate_validity_days` | Yes      | `5475`                    | Requested validity period for the new certificate in days (e.g., 7, 30, 90, 365, 730, up to 5475 for Origin CA certs). |
| `webserver_service_name`       | No       | `nginx`                   | The name of the webserver service to be restarted by the handler (e.g., `nginx`, `apache2`, `httpd`).       |

**Important Security Note on `cf_api_token`**:
It is strongly recommended to store your `cf_api_token` in Ansible Vault to keep it secure.

Example variable definition (e.g., in `host_vars/your_server.yml` or `group_vars/all.yml`):

```yaml
cf_api_token: "your_super_secret_cloudflare_api_token"
cf_zone_id: "your_zone_id_from_cloudflare"
cf_certificate_hostnames:
  - "example.com"
  - "*.example.com"
cf_certificate_install_path: "/etc/nginx/ssl/cloudflare/"
cf_certificate_file: "{{ cf_certificate_install_path }}origin.pem"
cf_key_file: "{{ cf_certificate_install_path }}origin.key"
cf_certificate_validity_days: 5475 # Request a 15-year certificate
webserver_service_name: "nginx"
```

## Cloudflare API Token and Zone ID

### 1. Generating a Cloudflare API Token

For optimal security, create a custom API Token with limited permissions instead of using your Global API Key.

**Steps to create a scoped API Token:**

1.  Log in to your Cloudflare dashboard.
2.  Go to **My Profile** > **API Tokens** (or click [here](https://dash.cloudflare.com/profile/api-tokens)).
3.  Click **Create Token**.
4.  You can start with a template or create a custom token. For this role, a **Custom token** is recommended. Click **Get started**.
5.  Give your token a descriptive name (e.g., "Ansible Origin Cert Management").
6.  Under **Permissions**, configure the following:
    * Select **Zone** as the resource type.
    * Select **SSL and Certificates** as the permission group.
    * Select **Edit** as the permission level. (The "Edit" permission typically includes read access needed to check existing certificates and write access to create new ones).
7.  Under **Zone Resources**:
    * Select **Include**.
    * Choose **Specific zone**.
    * Select the zone(s) you want this token to have access to. **It is crucial to limit this to only the zones this Ansible role will manage.**
8.  (Optional) You can also configure Client IP Address Filtering for added security if your Ansible control node has a static IP.
9.  Click **Continue to summary**.
10. Review the summary and click **Create Token**.
11. **Important**: Cloudflare will display the token secret **only once**. Copy it immediately and store it securely, preferably using Ansible Vault.

For more detailed information, refer to the official Cloudflare documentation:
* [Creating Cloudflare API tokens](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
* [API token permissions](https://developers.cloudflare.com/fundamentals/api/reference/permissions/) (look for "Zone SSL and Certificates")

### 2. Obtaining Your Cloudflare Zone ID

The Zone ID is a unique identifier for your domain (zone) within Cloudflare.

**Steps to find your Zone ID:**

1.  Log in to your Cloudflare dashboard.
2.  Select the domain (zone) for which you want to manage Origin Certificates.
3.  On the **Overview** page for that domain, scroll down the right-hand sidebar.
4.  You will find the **Zone ID** listed there. Click to copy it.

Alternatively, you can find it via the API or other methods described in the official documentation:
* [Finding your Cloudflare Account and Zone IDs](https://developers.cloudflare.com/fundamentals/setup/find-account-and-zone-ids/)

## Dependencies

* This role relies on standard `ansible.builtin` modules. No special collections need to be installed beyond a functioning Ansible installation.
* The target node requires a user with sufficient privileges to write files to the specified installation paths and restart the webserver.

## Example Playbook

Here's a basic example of how to use this role in your playbook:

```yaml
---
- name: Configure Web Server and Manage Cloudflare Origin Certificate
  hosts: web
  become: yes # Required for installing files in system paths and restarting services

  vars:
    cf_api_token: "vault_cf_api_token"
    cf_zone_id: "your_actual_zone_id"
    cf_certificate_hostnames:
      - "lbajsarowicz.cloud"
      - "*.lbajsarowicz.cloud"
    cf_certificate_install_path: "/etc/ssl/certs/cloudflare"
    cf_certificate_file: "{{ cf_certificate_install_path }}/origin.pem"
    cf_key_file: "{{ cf_certificate_install_path }}/origin.key"
    cf_certificate_validity_days: 5475 # Max validity for Origin Certs
    webserver_service_name: "nginx" # or "apache2", etc.

  roles:
    - role: lbajsarowicz.cloudflare_origin_cert

  handlers:
    - name: Restart webserver
      ansible.builtin.service:
        name: "{{ webserver_service_name }}"
        state: restarted
      listen: Restart webserver # This must match the notify statement in the role
```

### Handlers

This role uses a `notify` statement to trigger a handler named `Restart webserver` after successfully installing new certificate files. You need to ensure this handler is defined in your playbook or in a `handlers/main.yml` file accessible to your playbook (as shown in the example above, or within the role itself if you prefer).
