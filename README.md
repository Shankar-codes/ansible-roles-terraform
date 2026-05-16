# Ansible Roles — Roboshop Infrastructure Automation

A production-grade Ansible project that automates the provisioning and configuration of the **Roboshop** e-commerce application stack on AWS. It handles both infrastructure creation (EC2 + Route 53) and full application configuration through reusable Ansible roles.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Usage](#usage)
  - [1. Provision Infrastructure](#1-provision-infrastructure)
  - [2. Configure Application Components](#2-configure-application-components)
- [Inventory](#inventory)
- [Vault & Secrets](#vault--secrets)
- [Ansible Configuration](#ansible-configuration)
- [Security Considerations](#security-considerations)

---

## Overview

This repository contains Ansible automation for deploying the Roboshop microservices application. The project is split into two phases:

1. **Infrastructure provisioning** — spins up EC2 instances on AWS and registers them in Route 53 under the `ellamma.fun` domain.
2. **Configuration management** — applies component-specific Ansible roles (MongoDB, Redis, MySQL, RabbitMQ, Catalogue, Cart, User, Shipping, Payment, Frontend) to the newly created instances.

---

## Architecture

```
                          ┌──────────────────────────────────────┐
                          │           AWS Infrastructure          │
                          │                                       │
  Internet  ──────────►  │  frontend.ellamma.fun  (Public IP)    │
                          │         │                             │
                          │  ┌──────▼──────────────────────┐     │
                          │  │     Internal Services        │     │
                          │  │  (Private IP, DNS via R53)   │     │
                          │  │                              │     │
                          │  │  catalogue  cart  user       │     │
                          │  │  shipping   payment          │     │
                          │  │                              │     │
                          │  │  mongodb  mysql  redis       │     │
                          │  │  rabbitmq                    │     │
                          │  └──────────────────────────────┘     │
                          └──────────────────────────────────────┘
```

- All **backend services** use **private IP addresses** registered in Route 53.
- The **frontend** instance is registered with its **public IP** as the apex `ellamma.fun` A record.
- All EC2 instances use the `t3.micro` instance type.

---

## Project Structure

```
ansible-roles/
├── ansible.cfg                      # Ansible runtime configuration
├── inventory.ini                    # Static inventory of all hosts
├── main.yaml                        # Main playbook (component-driven)
├── create-ec2-r53-ansible.yaml      # Infrastructure provisioning playbook
├── mysql_vault_pass.txt             # Ansible Vault password file (do not commit plaintext)
├── group_vars/                      # Group-level variable files
│   └── <component>.yaml            # Per-component variables (e.g. mongodb.yaml)
└── roles/                           # Ansible roles
    ├── mongodb/
    ├── redis/
    ├── mysql/
    ├── rabbitmq/
    ├── catalogue/
    ├── cart/
    ├── user/
    ├── shipping/
    ├── payment/
    └── frontend/
```

---

## Components

| Component  | Host                        | Description                              |
|------------|-----------------------------|------------------------------------------|
| MongoDB    | mongodb.ellamma.fun         | Primary document store                   |
| Redis      | redis.ellamma.fun           | Session / caching layer                  |
| MySQL      | mysql.ellamma.fun           | Relational database (shipping/users)     |
| RabbitMQ   | rabbitmq.ellamma.fun        | Message broker for async communication   |
| Catalogue  | catalogue.ellamma.fun       | Product catalogue microservice (Node.js) |
| Cart       | cart.ellamma.fun            | Shopping cart microservice (Node.js)     |
| User       | user.ellamma.fun            | User management microservice (Node.js)   |
| Shipping   | shipping.ellamma.fun        | Shipping calculation microservice (Java) |
| Payment    | payment.ellamma.fun         | Payment processing microservice (Python) |
| Frontend   | frontend.ellamma.fun        | Nginx-based web frontend (public-facing) |

---

## Prerequisites

- **Ansible** ≥ 2.12
- **Python** ≥ 3.8
- **boto3** and **botocore** (for AWS modules)
- AWS credentials configured (`~/.aws/credentials` or environment variables)
- SSH access to EC2 instances (key pair configured on the instances)
- The `amazon.aws` collection:

```bash
ansible-galaxy collection install amazon.aws
```

---

## Configuration

### `ansible.cfg`

```ini
[defaults]
fork=2
inventory=./inventory.ini
log_path=/var/log/ellamma/ansible.log
vault_password_file=/etc/ansible/mysql_vault_pass.txt
```

| Setting               | Description                                              |
|-----------------------|----------------------------------------------------------|
| `fork=2`              | Run 2 tasks in parallel                                  |
| `inventory`           | Path to the static inventory file                        |
| `log_path`            | Where Ansible logs are written                           |
| `vault_password_file` | Path to the file containing the Ansible Vault passphrase |

---

## Usage

### 1. Provision Infrastructure

Creates EC2 instances and registers them in Route 53. Pass the list of components to create via the `instance` variable:

```bash
ansible-playbook create-ec2-r53-ansible.yaml \
  -e "instance=['mongodb','redis','mysql','rabbitmq','catalogue','cart','user','shipping','payment','frontend']"
```

**What it does:**

- Launches `t3.micro` EC2 instances (AMI: `ami-0220d79f3f480ecf5`) for each item in `instance`.
- Creates Route 53 A records under `ellamma.fun`:
  - All services → private IP address.
  - `frontend` → public IP address (apex domain record).

### 2. Configure Application Components

Run a role against a specific component using the `component` variable:

```bash
# Configure MongoDB
ansible-playbook main.yaml -e "component=mongodb"

# Configure Redis
ansible-playbook main.yaml -e "component=redis"

# Configure the frontend
ansible-playbook main.yaml -e "component=frontend"
```

The `main.yaml` playbook targets the host group matching the component name and applies the corresponding role from the `roles/` directory.

**To configure all components in sequence:**

```bash
for component in mongodb redis mysql rabbitmq catalogue cart user shipping payment frontend; do
  ansible-playbook main.yaml -e "component=$component"
done
```

---

## Inventory

All hosts are grouped by component in `inventory.ini`. Global SSH credentials are set under `[all:vars]`:

```ini
[mongodb]
mongodb.ellamma.fun

[catalogue]
catalogue.ellamma.fun

# ... (one group per component)

[all:vars]
ansible_user=ec2-user
ansible_password=DevOps321
```

> **⚠️ Note:** Storing plaintext passwords in `inventory.ini` is not recommended for production. Use Ansible Vault or SSH key-based authentication instead.

---

## Vault & Secrets

Sensitive values (e.g. MySQL root password) are encrypted with Ansible Vault. The vault password is read from the path defined in `ansible.cfg`:

```
vault_password_file=/etc/ansible/mysql_vault_pass.txt
```

To encrypt a variable:

```bash
ansible-vault encrypt_string 'mysecretvalue' --name 'mysql_root_password'
```

To view an encrypted file:

```bash
ansible-vault view group_vars/mysql.yaml
```

---

## Security Considerations

- **Do not commit** `mysql_vault_pass.txt` or any plaintext secrets to version control. Add it to `.gitignore`.
- Replace `ansible_password` in `inventory.ini` with SSH key-based authentication for production environments.
- Restrict the security group (`sg-020b146d0d1696186`) to only the necessary ports for each service.
- Rotate IAM credentials used by the provisioning playbook regularly.
- Consider using AWS Secrets Manager or HashiCorp Vault for secret management at scale.
