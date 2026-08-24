# Automação Ansible para Estações de Trabalho e Infraestrutura

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
![Licença](https://img.shields.io/badge/License-MIT-green.svg)

Repositório de automação de infraestrutura contendo roles e playbooks modulares do Ansible para provisionamento de base do SO, hardening de segurança CIS, Docker Engine, Jenkins CI e orquestração de clusters Kubernetes nativos (v1.36). Projetado para implantação em nós **Oracle Linux 9 (OL9)**, **RHEL** e **Debian/Ubuntu** em provedores de nuvem (OCI, AWS, GCP, Azure) ou VMs locais.

---

## 📁 Estrutura do Repositório

```text
deploy/
├── ansible.cfg.example          # Modelo global de configuração do Ansible
├── ssh.config.example           # Modelo de configuração SSH JumpHost / ProxyJump
├── site.yml                     # 🛡️ Playbook Principal (Orquestrador Master)
├── env/
│   └── hosts.example            # Modelo de Inventário Genérico (Copiar para env/hosts)
└── roles/
    ├── k8s-node-setup/          # ⚙️ Preparação do SO, módulos de kernel, containerd, binários K8s v1.36
    ├── k8s-control-plane/       # ☸️ kubeadm init, Flannel CNI, ferramentas CLI exclusivas do Master
    ├── k8s-worker/              # 🔗 Execução do kubeadm join & rotulagem de nós workers
    ├── base-packages/           # Utilitários do sistema e SRE essenciais
    ├── docker/                  # Provisionamento do Docker Engine & Docker Compose
    ├── jenkins/                 # Automação do Servidor Jenkins CI
    └── security-hardening/      # Hardening de Segurança CIS Benchmark
```

---

## 🛠️ Guia de Configuração Genérico

Siga estes passos para configurar e executar os playbooks em qualquer ambiente (local, nuvem ou híbrido):

### Passo 1: Instalar o Ansible & Pré-requisitos

Na sua máquina de controle local (Linux / WSL2):

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip sshpass
python3 -m venv pyenv
source pyenv/bin/activate
pip install --upgrade pip
pip install ansible
ansible --version
```

### Passo 2: Inicializar Modelos de Configuração

Navegue até o diretório `deploy/` e copie os modelos de configuração de exemplo:

```bash
cd deploy/
cp ansible.cfg.example ansible.cfg
cp ssh.config.example ssh.config
cp env/hosts.example env/hosts
```

### Passo 3: Preencher o Inventário (`deploy/env/hosts`)

Edite o arquivo `deploy/env/hosts` com seus endereços IP alvo e o caminho da chave SSH:

```ini
[bastion]
srv-bst-01 ansible_host=<SEU_IP_PUBLICO_BASTION>

[master]
srv-k8s-01 ansible_host=<SEU_IP_MASTER> internal_ip=<SEU_IP_MASTER>

[workers]
srv-k8s-02 ansible_host=<SEU_IP_WORKER1> internal_ip=<SEU_IP_WORKER1>
srv-k8s-03 ansible_host=<SEU_IP_WORKER2> internal_ip=<SEU_IP_WORKER2>

[k8s_cluster:children]
master
workers

[k8s_cluster:vars]
# Opcional: Configuração de ProxyJump se os nós alvo estiverem em uma rede privada
ansible_ssh_common_args='-o ProxyJump=opc@<SEU_IP_PUBLICO_BASTION> -o StrictHostKeyChecking=no'

[all:vars]
ansible_user=opc
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

### Passo 4: Verificar Conectividade SSH

Teste a conectividade para todos os alvos do inventário:

```bash
ansible all -i env/hosts -m ping
```

---

## 🚀 Como Executar os Playbooks

### 1. Playbook Principal (`deploy/site.yml`)

O playbook `site.yml` é o orquestrador master. Ele executa a pipeline de infraestrutura completa sequencialmente:

```bash
ansible-playbook site.yml -i env/hosts
```

#### Execução Seletiva via Tags:
Você pode executar roles ou componentes específicos usando tags do Ansible:

- **Apenas Hardening de Segurança**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags security
  ```
- **Apenas Pacotes Base & Utilitários SRE**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags base
  ```
- **Apenas Docker Engine**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags docker
  ```
- **Apenas Servidor Jenkins CI**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags jenkins
  ```
- **Apenas Cluster Kubernetes**:
  ```bash
  ansible-playbook site.yml -i env/hosts --tags k8s
  ```

#### O que o `site.yml --tags k8s` realiza (Arquitetura de 3 Plays):

1. **PLAY 1 (`k8s-node-setup`) - Todos os Nós (Master & Workers)**:
   - Habilita módulos de kernel (`overlay`, `br_netfilter`) e encaminhamento de rede sysctl.
   - Desabilita memória SWAP (requisito do K8s).
   - Instala & configura o runtime de contêiner `containerd` (`SystemdCgroup = true`).
   - Instala pacotes do sistema DNF incluindo **cliente `mysql`**, `nodejs 20 LTS` e a CLI `btop`.
   - Instala os binários do Kubernetes **v1.36** (`kubelet`, `kubeadm`, `kubectl`).
2. **PLAY 2 (`k8s-control-plane`) - Apenas Nó Master**:
   - Executa `kubeadm init` no nó Control Plane.
   - Configura `/home/opc/.kube/config`.
   - Implanta o plugin de rede CNI **Flannel**.
   - Gera dinamicamente tokens do `kubeadm join`.
   - **Instala Ferramentas CLI de DevOps e Gerenciamento de Banco de Dados Exclusivas do Master**:
     - `k9s` (Interface TUI para Kubernetes)
     - `argocd` (CLI do Argo CD)
     - `helm` (CLI do Helm)
     - `yq` (Processador de YAML)
     - `kubectx` & `kubens` (Alternadores de Contexto e Namespace)
     - `mongosh` (Shell do MongoDB)
     - `postgresql` (Cliente CLI para PostgreSQL)
3. **PLAY 3 (`k8s-worker`) - Apenas Nós Workers**:
   - Associa os nós Workers ao Control Plane usando os tokens gerados.
   - Rotula os nós Workers com `node-role.kubernetes.io/worker=worker`.

---

## 💡 Acesso Local via `kubectl` (Dica Pro)

Para gerenciar seu cluster Kubernetes diretamente da sua estação de controle local (WSL / Linux) sem precisar acessar o nó Master via SSH:

### 1. Baixar o `kubeconfig` do Nó Master
```bash
mkdir -p ~/.kube
ssh -F deploy/ssh.config srv-k8s-01 "cat ~/.kube/config" > ~/.kube/config
chmod 600 ~/.kube/config
```

### 2. Estabelecer Túnel SSH para Acesso Privado ao API Server
Se o seu nó Master estiver em uma sub-rede privada atrás de um Bastion Jump Host, redirecione a porta da API `6443` para a sua máquina local:
```bash
# Iniciar túnel SSH em segundo plano através do host Bastion
ssh -f -N -L 6443:10.0.2.11:6443 -F deploy/ssh.config oci-bastion
```

### 3. Atualizar Endereço do Servidor Local & Verificar Acesso
Atualize o endpoint do servidor do cluster para apontar para `localhost:6443`:
```bash
kubectl config set-cluster kubernetes --server=https://127.0.0.1:6443 --insecure-skip-tls-verify=true
kubectl get nodes -o wide
```

---

## 🚨 Solução de Problemas & Dicas

### 1. `kubectl` trava ou expira o tempo limite na Estação Local
* **Sintoma**: Executar `kubectl get nodes` na sua estação local (WSL/Linux) trava ou retorna `dial tcp 10.0.2.11:6443: connect: connection refused`.
* **Causa**: Endereços IP privados (`10.0.2.11`) dentro de uma VCN da nuvem não podem ser alcançados diretamente a partir de redes domésticas/corporativas sem um túnel SSH.
* **Solução**: Estabeleça um túnel SSH em segundo plano através do host Bastion e configure o endpoint do cluster local:
  ```bash
  ssh -f -N -L 6443:10.0.2.11:6443 -F deploy/ssh.config oci-bastion
  kubectl config set-cluster kubernetes --server=https://127.0.0.1:6443 --insecure-skip-tls-verify=true
  ```

### 2. `kubectl get nodes` retorna `localhost:8080 connection refused` nos nós Worker
* **Sintoma**: Executar comandos `kubectl` dentro de um nó Worker (`srv-k8s-02` ou `srv-k8s-03`) retorna `The connection to the server localhost:8080 was refused`.
* **Causa**: O API Server do Kubernetes (`kube-apiserver`) e as credenciais de admin (`~/.kube/config`) existem **apenas no nó Master (`srv-k8s-01`)**. Os Workers executam apenas cargas de trabalho de contêiner e, por segurança, não recebem chaves de admin do cluster.
* **Solução**: Execute comandos administrativos do `kubectl` no nó Master (`ssh srv-k8s-01`) ou da sua máquina local via o túnel SSH configurado acima.

### 3. Erros `UNREACHABLE` de SSH no Ansible via ProxyJump
* **Sintoma**: O Ansible falha com `UNREACHABLE!` ao conectar a nós privados (`10.0.2.11`, `10.0.2.12`, `10.0.2.13`).
* **Causa**: Definir `ansible_ssh_common_args` no inventário sobrescreve o `-F ./ssh.config` e remove os caminhos de chave SSH personalizados do jump host.
* **Solução**: Utilize o `deploy/ssh.config` (configurado em `deploy/ansible.cfg`), que gerencia nativamente `HostName`, `IdentityFile` e `ProxyJump`.

---

## 📄 Licença

Distribuído sob a [Licença MIT](LICENSE).
