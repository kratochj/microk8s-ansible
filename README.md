# Ansible Playbook for Configuring a Server with MicroK8s and Deploying Kubernetes Manifests

This Ansible playbook automates the process of configuring a Linux server to:

- Enable swap space
- Install MicroK8s (a lightweight Kubernetes distribution)
- Create Kubernetes namespaces and secrets
- Deploy Kubernetes manifests with domain customization
- Install necessary dependencies and tools

## Table of Contents

- [Prerequisites](#prerequisites)
- [Features](#features)
- [Directory Structure](#directory-structure)
- [Variables](#variables)
- [Usage](#usage)
- [Playbook Breakdown](#playbook-breakdown)
- [Notes](#notes)
- [License](#license)
- [Author](#author)

## Prerequisites

- **Ansible**: Version 2.9 or higher is recommended.
- **Control Machine Requirements**:
    - Python 3 installed.
    - Necessary Ansible collections:
        - `community.kubernetes`
- **Target Hosts**:
    - Debian/Ubuntu-based Linux distribution.
    - SSH access with `root` privileges.
    - Network connectivity to download packages and access repositories.

## Features

- **Swap Space Configuration**: Enables 2GB of swap space if not already present.
- **MicroK8s Installation**: Installs MicroK8s via Snap package manager.
- **Kubernetes Namespace Creation**: Creates namespaces specified in `namespaces.yaml`.
- **Secret Management**: Generates Kubernetes secrets with random tokens/passwords.
- **Manifest Deployment**: Processes and deploys Kubernetes manifests with domain name replacement.
- **DigitalOcean Agent Installation**: Conditionally installs the DigitalOcean monitoring agent.
- **Idempotency**: Designed to be safely run multiple times without causing issues.

## Directory Structure

```
microk8s-ansible/
├── playbook.yaml
├── namespaces.yaml
├── deployments/
│   └── *.yaml
└── README.md
```

- **`playbook.yaml`**: The main Ansible playbook.
- **`namespaces.yaml`**: Configuration file listing the Kubernetes namespaces to create.
- **`testlab_deployments/`**: Directory containing Kubernetes manifest files to deploy.
- **`README.md`**: Documentation for the playbook (this file).

## Variables

### Playbook Variables

- **`domain_name`**: Your domain name to replace in manifests (default: `"lab1.kratochvil.cloud"`).
- **`admin_token`**: Randomly generated 32-character admin token.
- **`mysql_root_password`**: Randomly generated 16-character MySQL root password.
- **`ansible_python_interpreter`**: Path to the Python 3 interpreter on the target hosts (default: `"/usr/bin/python3"`).

### Variables Loaded from Files

- **`namespaces.yaml`**: Contains the list of Kubernetes namespaces to create.

  ```yaml
  ---
  namespaces:
    - bitwarden
    - blog
    - monitoring
    - another-namespace
  ```

### Auto-Generated Variables

- **`swap_file`**: Checks if the swap file exists on the target host.
- **`fstab_backup`**: Checks if a backup of `/etc/fstab` exists.

## Usage

### 1. Clone or Download the Repository

```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
```

### 2. Install Required Ansible Collections

```bash
ansible-galaxy collection install community.kubernetes
```

### 3. Customize Variables

Edit the `playbook.yaml` file to set your domain name or override variables at runtime.

### 4. Prepare the Inventory File

Copy file `hosts-sample` to `hosts` and edit your inventory file by adding your target servers:

```ini
[all]
your_server_ip_or_hostname
```

### 5. Edit your ingress domain

Define your domain uder the ingress is deployed (i.e. ` lab.kratochvil.cloud`) . Your ingresses will be then exposed on `*.lab.kratochvil.eu`.

### 6. Define namespaces

If you need to create namespaces before to deployment scripts are executed (for example to create secrets in those namespaces), define them in the file `namespaces.yaml`.

### 5. Run the Playbook

Execute the playbook with Ansible:

```bash
ansible-playbook -i hosts.ini playbook.yaml
```

**Optional**: Override variables at runtime:

```bash
ansible-playbook -i hosts.ini playbook.yaml -e "domain_name=yourdomain.com"
```

### 6. Verify Deployment

After the playbook completes:

- Check that swap space is enabled: `swapon --show`
- Verify MicroK8s installation: `microk8s status --wait-ready`
- List Kubernetes namespaces: `microk8s kubectl get namespaces`
- Verify secrets: `microk8s kubectl get secrets -n your-namespace`

## Playbook Breakdown

### 1. **Swap Space Configuration**

- **Checks for existing swap file** and creates a 2GB swap file if not present.
- **Sets appropriate permissions** for the swap file.
- **Configures swap space** by updating `/etc/fstab` and enabling the swap.

### 2. **DigitalOcean Agent Installation**

- **Checks if the DigitalOcean agent (`do-agent`) is active**.
- **Installs the agent** only if it is not already running.

### 3. **Dependency Installation**

- **Installs `python3`, `python3-pip`, and `python3-kubernetes`** on Debian/Ubuntu systems.
- **Ensures `snapd` is installed** for Snap package management.

### 4. **MicroK8s Installation and Configuration**

- **Installs MicroK8s** via Snap.
- **Waits for MicroK8s to be ready**.
- **Enables essential MicroK8s addons**: `cert-manager`, `helm3`, `ingress`, etc.

### 5. **Kubernetes Namespace Creation**

- **Loads namespaces from `namespaces.yaml`**.
- **Creates each namespace** using the `k8s` module.

### 6. **Secret Management**

- **Generates secrets** for applications like Vaultwarden and MySQL.
- **Checks for existing secrets** to avoid overwriting them.
- **Uses random tokens/passwords** generated at runtime.

### 7. **Manifest Processing and Deployment**

- **Creates a temporary directory** on the control machine.
- **Copies manifests to the temporary directory**.
- **Replaces the domain name** in the manifests with the specified `domain_name`.
- **Copies the processed manifests to the target server**.
- **Applies the Kubernetes manifests** using `kubectl`.
- **Cleans up the temporary directory**.

## Notes

- **Control Machine**: Some tasks are delegated to the control machine (localhost). Ensure you have the necessary permissions and dependencies installed locally.
- **Privilege Escalation**: The playbook uses `become: yes` for tasks that require elevated privileges on the target host.
- **Idempotency**: The playbook is designed to be idempotent; it can be run multiple times without causing unintended changes.
- **Ansible Version**: Ensure you're using a compatible version of Ansible that supports all the modules and features used.
- **Sensitive Data**: Secrets are generated and used within the playbook. Handle with care and consider using Ansible Vault for added security.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE.md) file for details.

## Author

[Jiri Kratochvil](https://github.com/kratochj)

Feel free to contribute or raise issues on GitHub.