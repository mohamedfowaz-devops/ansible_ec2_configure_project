# Ansible EC2 Configuration Project

## 📌 Overview

Automated configuration of multiple AWS EC2 instances using **Ansible** from a centralized Control Node.

The project demonstrates:

* Ansible Inventory & Playbooks
* Roles and Variables
* Jinja2 Templates
* Ansible Vault
* SSH authentication
* Docker & Nginx configuration
* Idempotency
* Privilege escalation

## 🏗️ Architecture

```text
Windows Laptop
      |
      | SSH
      ↓
Ansible Control EC2
      |
      | Private SSH
      ↓
 ┌───────────┐
 │           │
Web-01      Web-02
 EC2         EC2
```

## 🛠️ Technologies

* AWS EC2
* Ansible
* Ubuntu/Linux
* Docker
* Nginx
* Git & GitHub
* SSH

## 📂 Project Structure

```text
ansible-ec2-configuration/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
│   └── all/
│       ├── vars.yml
│       └── vault.yml
└── roles/
    ├── common/
    ├── docker/
    ├── flask/
    └── nginx/
```

## ⚙️ Configuration

### Inventory

```ini
[webservers]

web01 ansible_host=172.31.1.42
web02 ansible_host=172.31.14.24
```

### Playbook

```yaml
---
- name: Configure EC2 web servers
  hosts: webservers
  become: true

  roles:
    - common
    - docker
    - flask
    - nginx
```

## 🚀 Usage

Check inventory:

```bash
ansible-inventory --graph
```

Test connectivity:

```bash
ansible all -m ping
```

Validate syntax:

```bash
ansible-playbook site.yml --syntax-check
```

Run playbook:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Check Docker:

```bash
ansible webservers -m command -a "docker --version"
```

Check Nginx:

```bash
ansible webservers -m shell -a "systemctl is-active nginx"
```

## 🔐 Security

Sensitive files are excluded using `.gitignore`:

```gitignore
*.pem
*.key
group_vars/all/vault.yml
```

Ansible Vault is used to protect sensitive variables.

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

## 🔄 Idempotency

Running the playbook multiple times does not unnecessarily change already-configured servers.

```bash
ansible-playbook site.yml --ask-vault-pass
```

## 🎯 Key Learning

This project provides hands-on experience with **Ansible automation, AWS EC2 configuration, roles, templates, Vault, SSH, Docker, Nginx, and idempotent infrastructure management.**


AWS DevOps | AWS | Terraform | Docker | Kubernetes | Jenkins | Ansible | Linux
