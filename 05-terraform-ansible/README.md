# 🏗️ Terraform & Ansible

## Two Tools, One Goal: Reproducible Infrastructure

| Tool | What it does | When to use |
|------|-------------|-------------|
| **Terraform** | Provision cloud resources (VMs, VPCs, DBs) | Creating/destroying infra |
| **Ansible** | Configure what's already provisioned | Installing software, users, configs |

> **Golden combo (used at Colpari):** Terraform creates Linux VMs → Ansible provisions them with services, users, and systemd configs — zero manual SSH.

---

## 🏗️ Terraform

### Core Concepts
```
Providers → Resources → State → Plan → Apply
```

### Essential Commands
```bash
terraform init              # download providers & modules
terraform fmt               # auto-format code
terraform validate          # check syntax
terraform plan -out=tfplan  # preview + save plan
terraform apply tfplan      # apply saved plan
terraform destroy           # tear down all resources

# State
terraform state list
terraform state show aws_instance.web
terraform import aws_instance.web i-1234abcd

# Workspaces (dev / staging / prod)
terraform workspace new staging
terraform workspace select staging
```

### Real-World: VPC + EC2 with Remote State
```hcl
# versions.tf
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}

# main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"
  name    = "${var.env}-vpc"
  cidr    = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = var.env != "prod"
  tags = local.common_tags
}

locals {
  common_tags = {
    Environment = var.env
    ManagedBy   = "Terraform"
    Team        = "DevOps"
  }
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = module.vpc.private_subnets[count.index % length(module.vpc.private_subnets)]
  tags = merge(local.common_tags, { Name = "${var.env}-web-${count.index + 1}" })
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter { name = "name"; values = ["amzn2-ami-hvm-*-x86_64-gp2"] }
}
```

---

## 📋 Ansible

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Inventory** | List of hosts to manage |
| **Playbook** | YAML describing what to do on which hosts |
| **Role** | Reusable bundle of tasks, handlers, templates |
| **Task** | Single action (install pkg, copy file, start service) |
| **Handler** | Task triggered only when notified (e.g. restart nginx) |
| **Vault** | Encrypted secrets |

### Cheatsheet
```bash
# Ad-hoc
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -m shell -a "df -h"

# Playbooks
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --tags "nginx"
ansible-playbook -i inventory.ini site.yml --limit "web-01"
ansible-playbook -i inventory.ini site.yml --check    # dry run
ansible-playbook -i inventory.ini site.yml -vvv       # verbose

# Vault
ansible-vault encrypt secrets.yml
ansible-vault edit secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

### Production Playbook (Colpari-style multi-node provisioner)
```yaml
# site.yml
---
- name: Provision web servers
  hosts: web
  become: yes
  vars_files:
    - vars/common.yml
    - vars/secrets.yml     # ansible-vault encrypted

  roles:
    - common
    - nginx
    - app-deploy

# roles/common/tasks/main.yml
---
- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install common packages
  apt:
    name: [curl, vim, htop, python3-pip, fail2ban]
    state: present

- name: Create deploy user
  user:
    name: deploy
    groups: sudo
    shell: /bin/bash
    create_home: yes

- name: Configure systemd service
  template:
    src: myapp.service.j2
    dest: /etc/systemd/system/myapp.service
    mode: '0644'
  notify: restart myapp
```

```ini
# inventory/production.ini
[web]
web-01 ansible_host=10.0.1.10
web-02 ansible_host=10.0.1.11

[db]
db-01 ansible_host=10.0.2.10

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/prod_key
```

---

## 🛠️ Hands-On Project: Zero-Touch Linux Provisioner

**Flow:** Terraform creates EC2 → outputs IPs → Ansible picks up and configures: users, packages, app service, firewall.

📁 See [`examples/`](./examples/) for full code.

---

## 🎯 Key Interview Questions

1. **Why use remote state with DynamoDB locking?**  
   S3 stores state shared across team. DynamoDB locks prevent concurrent `apply` from corrupting state.

2. **Difference between Ansible `copy` and `template` module?**  
   `copy` transfers a static file. `template` renders Jinja2 variables before copying — ideal for dynamic config files.

3. **How do you secure secrets in Ansible?**  
   Ansible Vault encrypts variable files. The vault password is stored as a CI/CD secret and passed via `--vault-password-file`.

4. **`count` vs `for_each` in Terraform?**  
   `count` creates indexed resources. `for_each` uses a map — safer because removing one item doesn't shift indexes and trigger unwanted destroy/recreate.

5. **Terraform vs Ansible — when to use which?**  
   Terraform for provisioning cloud resources (declarative, state-based). Ansible for configuring what's already running (procedural, agentless).
