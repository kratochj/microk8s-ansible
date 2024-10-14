# Ansible Playbook for Configuring a MicroK8s Cluster with Multiple Nodes and Deploying Kubernetes Manifests

This Ansible project automates the configuration of a MicroK8s Kubernetes cluster across multiple nodes, including:

- Configuring master and worker nodes
- Enabling swap space
- Installing necessary dependencies
- Setting up MicroK8s and enabling addons
- Joining worker nodes to the cluster
- Managing Kubernetes namespaces and secrets
- Deploying Kubernetes manifests with domain customization
- Installing the DigitalOcean Agent on all nodes

## Table of Contents

- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Features](#features)
- [Variables](#variables)
- [Usage](#usage)
- [Playbook Breakdown](#playbook-breakdown)
- [Roles](#roles)
- [Notes](#notes)
- [License](#license)
- [Author](#author)

## Project Structure

```
project-directory/
├── ansible.cfg
├── hosts.ini
├── playbook.yaml
├── roles/
│   ├── common/
│   │   └── tasks/
│   │       └── main.yml
│   ├── swap/
│   │   └── tasks/
│   │       └── main.yml
│   ├── microk8s/
│   │   └── tasks/
│   │       └── main.yml
│   ├── secrets/
│   │   └── tasks/
│   │       └── main.yml
│   ├── digitalocean_agent/
│   │   └── tasks/
│   │       └── main.yml
│   ├── k8s_namespaces/
│   │   └── tasks/
│   │       └── main.yml
├── group_vars/
│   ├── all.yml
│   ├── master.yml
│   ├── workers.yml
├── deployments/
│   └── *.yaml
└── README.md
```

## Prerequisites

- **Ansible**: Version 2.9 or higher is recommended.
- **Control Machine Requirements**:
  - Python 3 installed.
  - Necessary Ansible collections:
    - `community.kubernetes`
- **Target Hosts**:
  - Debian/Ubuntu-based Linux distribution.
  - SSH access with `root` privileges or appropriate sudo access.
  - Network connectivity to download packages and access repositories.

## Features

- **Swap Space Configuration**: Enables 2GB of swap space if not already present.
- **MicroK8s Installation**: Installs MicroK8s via Snap package manager on both master and worker nodes.
- **Cluster Formation**: Joins worker nodes to the master node's MicroK8s cluster.
- **Kubernetes Namespace Creation**: Creates namespaces specified in `namespaces.yaml`.
- **Secret Management**: Generates Kubernetes secrets with random tokens/passwords.
- **Manifest Deployment**: Processes and deploys Kubernetes manifests with domain name replacement.
- **DigitalOcean Agent Installation**: Installs the DigitalOcean monitoring agent on all nodes.
- **Idempotency**: Designed to be safely run multiple times without causing issues.

## Variables

### Playbook Variables

- **`domain_name`**: Your domain name to replace in manifests (default: `"lab2.kratochvil.cloud"`).
- **`admin_token`**: Randomly generated 32-character admin token.
- **`mysql_root_password`**: Randomly generated 16-character MySQL root password.
- **`ansible_python_interpreter`**: Path to the Python 3 interpreter on the target hosts (default: `"/usr/bin/python3"`).

### Group Variables

- **`group_vars/all.yml`**

  ```yaml
  domain_name: "lab2.kratochvil.cloud"
  ansible_python_interpreter: "/usr/bin/python3"
  ```

- **`group_vars/master.yml`**

  ```yaml
  admin_token: "{{ lookup('password', '/dev/null chars=ascii_letters,digits length=32') }}"
  mysql_root_password: "{{ lookup('password', '/dev/null chars=ascii_letters,digits length=16') }}"
  namespaces:
    - bitwarden
    - blog
    - monitoring
  kubernetes_secrets:
    - name: vaultwarden-admin-token
      namespace: bitwarden
      data:
        ADMIN_TOKEN: "{{ admin_token | b64encode }}"
    - name: mysql-secret
      namespace: blog
      data:
        MYSQL_ROOT_PASSWORD: "{{ mysql_root_password | b64encode }}"
  ```

- **`group_vars/workers.yml`**

  ```yaml
  # Define any variables specific to worker nodes here
  ```

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

Edit the group variable files (`group_vars/all.yml`, `group_vars/master.yml`) to set your domain name or other variables as needed.

### 4. Prepare the Inventory File

Create or edit your inventory file (`hosts.ini`) and add your target servers:

```ini
[master]
master-node ansible_host=MASTER_NODE_IP

[workers]
worker-node1 ansible_host=WORKER_NODE1_IP
worker-node2 ansible_host=WORKER_NODE2_IP

[all:vars]
ansible_user=root
```

Replace `MASTER_NODE_IP`, `WORKER_NODE1_IP`, and `WORKER_NODE2_IP` with the actual IP addresses of your nodes.

### 5. Run the Playbook

Execute the playbook with Ansible:

```bash
ansible-playbook -i hosts.ini playbook.yaml
```

### 6. Verify Deployment

After the playbook completes:

- **On the Master Node**:
  - Verify MicroK8s installation: `microk8s status --wait-ready`
  - List Kubernetes namespaces: `microk8s kubectl get namespaces`
  - Verify secrets: `microk8s kubectl get secrets -n your-namespace`
  - Check cluster nodes: `microk8s kubectl get nodes`
- **On Worker Nodes**:
  - Verify that MicroK8s is installed: `microk8s status --wait-ready`
  - Check that the node is part of the cluster from the master node.

## Playbook Breakdown

### 1. **Roles**

The playbook uses several roles to organize tasks:

- **`common`**: Installs common dependencies on all nodes (e.g., Python, snapd).
- **`swap`**: Configures swap space on all nodes.
- **`microk8s`**: Installs MicroK8s and enables necessary addons on master and worker nodes.
- **`digitalocean_agent`**: Installs the DigitalOcean monitoring agent on all nodes.
- **`k8s_namespaces`**: Creates Kubernetes namespaces on the master node.
- **`secrets`**: Manages Kubernetes secrets on the master node.

### 2. **Playbook Structure**

- **First Play** (`Configure All Nodes`):
  - **Hosts**: `all`
  - **Roles**: `common`, `swap`, `microk8s`, `digitalocean_agent`
  - **Tasks**:
    - **Master Node Tasks** (conditional):
      - Generate MicroK8s join token.
      - Fetch join command.
      - Create Kubernetes namespaces.
      - Apply secrets and manifests.
    - **Worker Node Tasks** (conditional):
      - Join worker nodes to the cluster (only if not already joined).
- **Second Play** (`Clean up join command on control machine`):
  - **Hosts**: `localhost`
  - **Tasks**:
    - Remove join command file from the control machine.

### 3. **Conditional Execution**

- **Using `when` Conditions**:
  - Tasks specific to the master node use `when: "'master' in group_names"`.
  - Tasks specific to worker nodes use `when: "'workers' in group_names"`.
  - The "Join the cluster" task on worker nodes is executed only if the node is not already part of the cluster.

### 4. **Idempotency**

- The playbook is designed to be idempotent. It checks for existing resources before creating them to avoid unintended changes.

## Roles

### **`common` Role**

- **Tasks**:
  - Installs `python3` and `python3-pip`.
  - Ensures `snapd` is installed.
  - Installs Kubernetes Python client.

### **`swap` Role**

- **Tasks**:
  - Checks if swap file exists.
  - Creates and enables a 2GB swap file if not present.
  - Backs up `/etc/fstab` and adds swap entry.

### **`microk8s` Role**

- **Tasks**:
  - Installs MicroK8s via snap.
  - Waits for MicroK8s to be ready.
  - **Master Node**:
    - Enables MicroK8s addons: `cert-manager`, `helm3`, `ingress`, etc.

### **`digitalocean_agent` Role**

- **Tasks**:
  - Checks if `do-agent` service is active.
  - Installs the DigitalOcean agent if not already running.

### **`k8s_namespaces` Role**

- **Tasks**:
  - Creates Kubernetes namespaces defined in `namespaces.yaml`.

### **`secrets` Role**

- **Tasks**:
  - Checks if Kubernetes secrets exist.
  - Creates secrets only if they do not already exist.

## Notes

- **Control Machine**: Some tasks are delegated to the control machine (localhost). Ensure you have the necessary permissions and dependencies installed locally.
- **Privilege Escalation**: The playbook uses `become: yes` for tasks that require elevated privileges on the target host.
- **Ansible Version**: Ensure you're using a compatible version of Ansible that supports all the modules and features used.
- **Sensitive Data**: Secrets are generated and used within the playbook. Handle with care and consider using Ansible Vault for added security.
- **Testing**: Test the playbook in a staging environment before deploying to production.
- **Idempotency**: Designed to be safely run multiple times without causing issues.

## License

This project is licensed under the MIT License. See the [LICENSE.md](LICENSE.md) file for details.

## Author

[Jiri Kratochvil](https://github.com/kratochj)

Feel free to contribute or raise issues on GitHub.