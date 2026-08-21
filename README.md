# Workstation & Infrastructure Ansible Automation

[![YAML Lint & Syntax Validation](https://github.com/aleknots/ansible/actions/workflows/yaml-lint.yml/badge.svg?branch=main)](https://github.com/aleknots/ansible/actions/workflows/yaml-lint.yml)
![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?logo=ansible&logoColor=white)
![Oracle Linux](https://img.shields.io/badge/Oracle%20Linux-9-F80000?logo=oracle&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20%7C%2026.04-E95420?logo=ubuntu&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-12-A80030?logo=debian&logoColor=white)
![RedHat](https://img.shields.io/badge/RedHat-RHEL%209-EE0000?logo=redhat&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.36-326CE5?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Engine-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![Security Hardening](https://img.shields.io/badge/CIS-Hardening-008080?logo=shield&logoColor=white)
![OCI](https://img.shields.io/badge/OCI-Always%20Free-F80000?logo=oracle&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green.svg)

Infrastructure automation repository containing modular Ansible roles and playbooks for OS baseline provisioning, CIS security hardening, Docker Engine, Jenkins CI, and native Kubernetes (v1.31) cluster orchestration. Designed for deployment on **Oracle Linux 9 (OL9)**, **RHEL**, and **Debian/Ubuntu** nodes across cloud providers (OCI, AWS, GCP, Azure) or local VMs.

---

## 📁 Repository Structure

```text
deploy/
├── ansible.cfg.example          # Global Ansible configuration template
├── ssh.config.example           # SSH JumpHost / ProxyJump configuration template
├── site.yml                     # 🛡️ Main Entrypoint Playbook (Master Orchestrator)
├── env/
│   └── hosts.example            # Generic Inventory template (Copy to env/hosts)
└── roles/
    ├── k8s-node-setup/          # ⚙️ OS prep, kernel modules, containerd, K8s v1.36 binaries
    ├── k8s-control-plane/       # ☸️ kubeadm init, Flannel CNI, Master-exclusive CLI tools
    ├── k8s-worker/              # 🔗 kubeadm join execution & worker node labeling
    ├── base-packages/           # Core SRE & system utilities
    ├── docker/                  # Docker Engine & Docker Compose provisioning
    ├── jenkins/                 # Jenkins CI Server automation
    └── security-hardening/      # CIS Benchmark Security Hardening
```

---

## 🛠️ Generic Setup Guide

Follow these steps to configure and run the playbooks in any environment (local, cloud, or hybrid):

### Step 1: Install Ansible & Prerequisites

On your local control machine (Linux / WSL2):

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip sshpass
python3 -m venv pyenv
source pyenv/bin/activate
pip install --upgrade pip
pip install ansible
ansible --version
```

### Step 2: Initialize Configuration Templates

Navigate to `deploy/` directory and copy example configuration templates:

```bash
cd deploy/
cp ansible.cfg.example ansible.cfg
cp ssh.config.example ssh.config
cp env/hosts.example env/hosts
```

### Step 3: Populate Inventory (`deploy/env/hosts`)

Edit `deploy/env/hosts` with your target IP addresses and SSH key path:

```ini
[bastion]
srv-bst-01 ansible_host=<YOUR_PUBLIC_BASTION_IP>

[master]
srv-k8s-01 ansible_host=<YOUR_MASTER_IP> internal_ip=<YOUR_MASTER_IP>

[workers]
srv-k8s-02 ansible_host=<YOUR_WORKER1_IP> internal_ip=<YOUR_WORKER1_IP>
srv-k8s-03 ansible_host=<YOUR_WORKER2_IP> internal_ip=<YOUR_WORKER2_IP>

[k8s_cluster:children]
master
workers

[k8s_cluster:vars]
# Optional: ProxyJump configuration if target nodes are in a private network
ansible_ssh_common_args='-o ProxyJump=opc@<YOUR_PUBLIC_BASTION_IP> -o StrictHostKeyChecking=no'

[all:vars]
ansible_user=opc
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

### Step 4: Verify SSH Connectivity

Test connectivity to all inventory targets:

```bash
ansible all -i env/hosts -m ping
```

---

## 🚀 How to Execute Playbooks

### 1. Main Entrypoint Playbook (`deploy/site.yml`)

The `site.yml` playbook is the master orchestrator. It executes the full end-to-end infrastructure pipeline sequentially:

```bash
ansible-playbook site.yml -i env/hosts
```

#### Selective Execution via Tags:
You can execute specific roles or components using Ansible tags:

- **Security Hardening Only**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags security
  ```
- **Base Packages & SRE Utilities Only**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags base
  ```
- **Docker Engine Only**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags docker
  ```
- **Jenkins CI Server Only**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags jenkins
  ```
- **Kubernetes Cluster Only**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags k8s
  ```

#### What `site.yml --tags k8s` Performs (3-Play Architecture):

1. **PLAY 1 (`k8s-node-setup`) - All Nodes (Master & Workers)**:
   - Enables kernel modules (`overlay`, `br_netfilter`) and sysctl network forwarding.
   - Disables SWAP memory (K8s requirement).
   - Installs & configures `containerd` container runtime (`SystemdCgroup = true`).
   - Installs DNF system packages (Opção 2) including **`mysql` client**, `nodejs 20 LTS`, and `btop` CLI.
   - Installs Kubernetes **v1.36** binaries (`kubelet`, `kubeadm`, `kubectl`).
2. **PLAY 2 (`k8s-control-plane`) - Master Node Only**:
   - Executes `kubeadm init` on Control Plane node.
   - Configures `/home/opc/.kube/config`.
   - Deploys **Flannel CNI** network plugin.
   - Dynamically generates `kubeadm join` tokens.
   - **Installs Master-Exclusive DevOps & DB Management CLI Tools**:
     - `k9s` (Kubernetes TUI)
     - `argocd` (Argo CD CLI)
     - `helm` (Helm CLI)
     - `yq` (YAML Processor)
     - `kubectx` & `kubens` (Context & Namespace Switchers)
     - `mongosh` (MongoDB Shell)
     - `postgresql` (PostgreSQL Client CLI)
3. **PLAY 3 (`k8s-worker`) - Worker Nodes Only**:
   - Joins Worker nodes to Control Plane using generated tokens.
   - Labels Worker nodes with `node-role.kubernetes.io/worker=worker`.

---

## 💡 Local `kubectl` Access (Pro-Tip)

To manage your Kubernetes cluster directly from your local control workstation (WSL / Linux) without needing to SSH into the Master node:

### 1. Download `kubeconfig` from Master Node
```bash
mkdir -p ~/.kube
ssh -F deploy/ssh.config srv-k8s-01 "cat ~/.kube/config" > ~/.kube/config
chmod 600 ~/.kube/config
```

### 2. Establish SSH Tunnel for Private API Server Access
If your Master node is in a private subnet behind a Bastion Jump Host, forward API port `6443` to your local machine:
```bash
# Start background SSH tunnel through Bastion host
ssh -f -N -L 6443:10.0.2.11:6443 -F deploy/ssh.config oci-bastion
```

### 3. Update Local Server Address & Verify Access
Update the cluster server endpoint to point to `localhost:6443`:
```bash
kubectl config set-cluster kubernetes --server=https://127.0.0.1:6443 --insecure-skip-tls-verify=true
kubectl get nodes -o wide
```

---

## 🚨 Troubleshooting & Gotchas

### 1. `kubectl` hangs or times out on Local Workstation
* **Symptom**: Running `kubectl get nodes` on your local WSL/Linux workstation hangs or returns `dial tcp 10.0.2.11:6443: connect: connection refused`.
* **Cause**: Private IP addresses (`10.0.2.11`) inside a Cloud VCN cannot be reached directly from home/office networks without an SSH tunnel.
* **Fix**: Establish a background SSH tunnel through the Bastion host and configure your local cluster endpoint:
  ```bash
  ssh -f -N -L 6443:10.0.2.11:6443 -F deploy/ssh.config oci-bastion
  kubectl config set-cluster kubernetes --server=https://127.0.0.1:6443 --insecure-skip-tls-verify=true
  ```

### 2. `kubectl get nodes` returns `localhost:8080 connection refused` on Worker nodes
* **Symptom**: Running `kubectl` commands inside a Worker node (`srv-k8s-02` or `srv-k8s-03`) returns `The connection to the server localhost:8080 was refused`.
* **Cause**: The Kubernetes API Server (`kube-apiserver`) and admin credentials (`~/.kube/config`) exist **only on the Master node (`srv-k8s-01`)**. Workers only run container workloads and are deliberately not granted cluster admin keys for security.
* **Fix**: Run administrative `kubectl` commands either on the Master node (`ssh srv-k8s-01`) or from your local machine via the SSH tunnel setup above.

### 3. Ansible SSH `UNREACHABLE` errors over ProxyJump
* **Symptom**: Ansible fails with `UNREACHABLE!` when connecting to private nodes (`10.0.2.11`, `10.0.2.12`, `10.0.2.13`).
* **Cause**: Defining `ansible_ssh_common_args` in inventory overrides `-F ./ssh.config` and drops custom SSH identity key paths for the jump host.
* **Fix**: Rely on `deploy/ssh.config` (configured in `deploy/ansible.cfg`), which natively manages `HostName`, `IdentityFile`, and `ProxyJump`.

---

## 📄 License

Distributed under the [MIT License](LICENSE).
