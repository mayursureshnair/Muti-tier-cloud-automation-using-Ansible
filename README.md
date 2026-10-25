# AWS Multi-Tier Cloud Automation with Ansible

An end-to-end infrastructure automation project demonstrating how to provision multi-tier AWS EC2 instances (both public and private subnets) using the AWS CLI, configure **passwordless SSH authentication**, and automate service provisioning using **Ansible playbooks** and **ad-hoc commands**.

---

## 📌 Architecture & Design Overview

```
                      +-------------------------------------------------------------------------+
                      |                          AWS VPC (10.0.0.0/16)                          |
                      |                           [ansible-lab-vpc]                             |
                      |                                                                         |
                      |  +-------------------------------------------------------------------+  |
                      |  |                Public Subnet (10.0.1.0/24)                        |  |
                      |  |                                                                   |  |
                      |  |   +-----------------------+        +--------------------------+   |  |
                      |  |   |  Ansible Controller   |        |       NAT Gateway        |   |  |
                      |  |   | (10.0.1.233, Public IP)        | (Enables outbound access)|   |  |
                      |  |   +-----------+-----------+        +------------+-------------+   |  |
                      |  |               |                                 ^                 |  |
                      |  |        Passwordless SSH                         | Outbound        |  |
                      |  |        (ed25519 keypair)                        | Internet        |  |
                      |  |               |                                 | Traffic         |  |
                      |  |        +------+------+                          | (apt packages)  |  |
                      |  |        |             |                          |                 |  |
                      |  |        v             v                          |                 |  |
                      |  |   +---------+   +---------+                     |                 |  |
                      |  |   | public1 |   | public2 |                     |                 |  |
                      |  |   |10.0.1.51|   |10.0.1.164                     |                 |  |
                      |  |   +---------+   +---------+                     |                 |  |
                      |  |   (Nginx, Git,  (Nginx, Git,                    |                 |  |
                      |  |    Docker)       Docker)                        |                 |  |
                      |  +----------------------+--------------------------+-----------------+  |
                      |                         |                          |                    |
                      |                         v SSH                      |                    |
                      |  +-------------------------------------------------+-----------------+  |
                      |  |                Private Subnet (10.0.2.0/24)     |                 |  |
                      |  |                [No Public IP]                   +                 |  |
                      |  |                                                                   |  |
                      |  |        +---------------------+                                    |  |
                      |  |        | private1            | (10.0.2.12)                        |  |
                      |  |        | [ansible-private]   +------------------------------------+  |
                      |  |        +---------------------+                                       |  |
                      |  |        (Nginx, PostgreSQL)                                           |  |
                      |  +-------------------------------------------------------------------+  |
                      +-------------------------------------------------------------------------+
```

### Key Highlights
- **Agentless Architecture:** Ansible requires zero custom agent daemons installed on managed target servers. Communication relies solely on standard OpenSSH and Python.
- **Dynamic AMI Lookup:** EC2 instances are provisioned with official Ubuntu 24.04 LTS AMIs dynamically resolved via AWS Systems Manager (SSM) Parameter Store.
- **NAT Gateway for Private Subnet Outbound Access:** A **NAT Gateway** attached to the public subnet allows instances in the private subnet (`private1`) to securely download packages and updates from the internet (`apt update`, installing `postgresql` and `nginx`) without assigning public IP addresses or exposing them to incoming internet traffic.
- **Dual-Tier Subnet Support:** Manages both internet-facing servers (public subnet) and isolated backend servers (private subnet with `--no-associate-public-ip-address`).
- **AWS Free-Tier Conscious:** Standard free-tier accounts offer a quota of 8 vCPUs. Using 4 `t3.micro` instances (2 vCPUs each) totals exactly 8 vCPUs.

---

## 📂 Project Structure

```
.
├── aws-cli-commands.txt       # AWS CLI commands for provisioning Controller, Public, & Private nodes
├── aws-resources-info.txt     # Subnet and Security Group IDs reference
├── inventory.ini              # Ansible inventory file grouping nodes into [public], [private], & [all]
├── first-playbook.yml         # Main playbook deploying Nginx, Docker, Git, & PostgreSQL
└── README.md                  # Comprehensive project documentation
```

---

## 🛠️ Step 1: AWS Infrastructure Provisioning via AWS CLI

All instances use Ubuntu 24.04 LTS resolved dynamically via SSM:
```
resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id
```

### 1.1 Launch Ansible Controller Node
```bash
aws ec2 run-instances \
  --region us-east-1 \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --instance-type t3.micro \
  --count 1 \
  --key-name ansible-lab-key \
  --subnet-id subnet-02dfff19452fc3458 \
  --security-group-ids sg-0d3194b7754a1a98a \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-controller}]'
```

### 1.2 Launch Public Target Instances
```bash
aws ec2 run-instances \
  --region us-east-1 \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --instance-type t3.micro \
  --count 2 \
  --key-name ansible-lab-key \
  --subnet-id subnet-02dfff19452fc3458 \
  --security-group-ids sg-084b0ff369087484e \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-public}]'
```

### 1.3 Launch Private Target Instance (No Public IP)
```bash
aws ec2 run-instances \
  --region us-east-1 \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --instance-type t3.micro \
  --count 1 \
  --key-name ansible-lab-key \
  --subnet-id subnet-0b582505c5ce175fe \
  --security-group-ids sg-084b0ff369087484e \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-private}]'
```

---

## 🔑 Step 2: Configuring Passwordless SSH Authentication

Ansible connects to remote machines over SSH. When initially attempting SSH connection from the Controller node to target nodes, authentication fails (`Permission denied (publickey)`).

### 2.1 Generate ED25519 Keypair on Ansible Controller
Log into the Controller (`10.0.1.233`) and generate a keypair without a passphrase:
```bash
ssh-keygen -t ed25519
```
*Key generated:* `~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`.

### 2.2 Distribute Public Key to All Target Nodes
View the public key on the Controller:
```bash
cat ~/.ssh/id_ed25519.pub
```

On each target server (`public1: 10.0.1.51`, `public2: 10.0.1.164`, `private1: 10.0.2.12`):
```bash
echo "<COPIED_ED25519_PUBLIC_KEY>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 2.3 Verify Passwordless Connectivity from Controller
```bash
ssh ubuntu@10.0.1.51
ssh ubuntu@10.0.1.164
ssh ubuntu@10.0.2.12
```
All targets connect instantly without prompting for passwords or key flags.

---

## ⚙️ Step 3: Ansible Installation & Configuration

On the Ansible Controller node:
```bash
sudo apt update && sudo apt install ansible -y
ansible --version
```

### 3.1 Inventory Configuration (`inventory.ini`)
Defines managed targets divided into groups and specifies connection user:

```ini
[public]
public1 ansible_host=10.0.1.51
public2 ansible_host=10.0.1.164

[private]
private1 ansible_host=10.0.2.12

[all:children]
public
private

[all:vars]
ansible_user=ubuntu
```

---

## 🚀 Step 4: Playbook Execution

The playbook [`first-playbook.yml`](first-playbook.yml) orchestrates services across different server tiers:
1. **All nodes (`all`):** Installs, enables, and starts **Nginx**.
2. **Public nodes (`public`):** Installs **Git** and **Docker**, then starts and enables Docker daemon.
3. **Private nodes (`private`):** Installs, enables, and starts **PostgreSQL**.

### Run the Playbook:
```bash
ansible-playbook -i inventory.ini first-playbook.yml
```

### Output Summary:
```text
PLAY RECAP *********************************************************************************
private1    : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
public1     : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
public2     : ok=6    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

---

## ⚡ Step 5: Verification & Ad-Hoc Commands

Ansible ad-hoc commands execute quick single tasks across one or more target groups without writing a playbook.

### 5.1 Playbook vs Ad-Hoc Commands
| Type | Best Used For | Execution Style |
| :--- | :--- | :--- |
| **Playbook** | Multi-step deployments, orchestration, repeatable configuration state | Declarative YAML file (`ansible-playbook`) |
| **Ad-Hoc Commands** | Quick checks, single package installations, one-off service restarts | Single CLI command (`ansible <group> -m <module> -a "<args>"`) |

### 5.2 Verification Ad-Hoc Commands

#### Check Git Version on Public Servers
```bash
ansible public -i inventory.ini -m command -a "git --version"
```
*Output: `git version 2.43.0` on `public1` and `public2`.*

#### Verify Nginx Service Status Across All Servers
```bash
ansible all -i inventory.ini -m shell -a "systemctl is-active nginx"
```
*Output: `active` on `public1`, `public2`, and `private1`.*

#### Check Docker Version on Public Servers
```bash
ansible public -i inventory.ini -m command -a "docker --version"
```
*Output: `Docker version 29.1.3` on `public1` and `public2`.*

#### Verify PostgreSQL Service on Private Server
```bash
ansible private -i inventory.ini -m shell -a "systemctl is-active postgresql"
```
*Output: `active` on `private1`.*

### 5.3 One-Off Package Installation via Ad-Hoc Command
To install `python3` on `public1` using elevated privileges (`-b` / become):
```bash
ansible public1 -i inventory.ini -b -m apt -a "name=python3 state=present update_cache=yes"
```

Verify Python installation:
```bash
ansible public1 -i inventory.ini -m command -a "python3 --version"
```
*Output: `Python 3.12.3`.*

---

## 💡 Key Takeaways
- **No Console Dependency:** Entire EC2 multi-instance environment is spin up rapidly with AWS CLI.
- **Key-Based Trust:** Simple and reliable SSH public key deployment ensures secure passwordless communication for Ansible.
- **NAT Gateway for Private Subnets:** Ensures isolated database servers can receive security updates and packages while preventing direct exposure to the public internet.
- **Group Targeting:** Tailored playbooks deploy specific applications (e.g. databases on private nodes, web servers across all nodes, dev tools on public nodes) in a single run.
- **Idempotency:** Re-running the playbook checks the current state and only applies changes if needed (`changed=0` on clean runs).
