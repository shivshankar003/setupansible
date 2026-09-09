# Raspberry Pi Ansible Setup

A simple guide to set up **Ansible on an Ubuntu VM** and use it to connect to and deploy applications on **Raspberry Pi** devices.

## Architecture

```text
Windows PC
   |
   v
Oracle VirtualBox
   |
   v
Ubuntu VM
   |
   | SSH + Ansible
   v
Raspberry Pi
   |
   v
Application Deployment
```

## 1. Create Ubuntu VM

Install Ubuntu in Oracle VirtualBox.

Recommended:

- RAM: 4 GB
- CPU: 2 cores or more
- Network: **Bridged Adapter**

Bridged Adapter allows the Ubuntu VM and Raspberry Pi to communicate on the same network.

Check the Ubuntu VM IP:

```bash
hostname -I
```

## 2. Install Ansible

Open the Ubuntu terminal:

```bash
sudo apt update
sudo apt install ansible openssh-client git -y
```

Check Ansible:

```bash
ansible --version
```

## 3. Connect to Raspberry Pi

Make sure SSH is enabled on the Raspberry Pi.

Find the Raspberry Pi IP:

```bash
hostname -I
```

From Ubuntu, test SSH:

```bash
ssh dreamvu@192.168.1.17
```

Replace the username and IP with your Raspberry Pi details.

## 4. Set Up SSH Key Authentication

On Ubuntu:

```bash
ssh-keygen
```

Copy the public key to the Raspberry Pi:

```bash
ssh-copy-id dreamvu@192.168.1.17
```

Test:

```bash
ssh dreamvu@192.168.1.17
```

If this works without asking for the Raspberry Pi password, SSH key authentication is ready.

## 5. Create Ansible Project

```bash
mkdir -p ~/ansible-raspberrypi
cd ~/ansible-raspberrypi
```

Recommended structure:

```text
ansible-raspberrypi/
├── README.md
├── inventory.ini
├── deploy.yml
└── .gitignore
```

## 6. Create Inventory

Create:

```bash
nano inventory.ini
```

Example:

```ini
[raspberrypi]
pi1 ansible_host=192.168.1.17

[raspberrypi:vars]
ansible_user=dreamvu
```

Replace the IP address and username with your Raspberry Pi details.

## 7. Test Ansible

Run:

```bash
ansible raspberrypi -i inventory.ini -m ping
```

Expected result:

```text
pi1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you get `pong`, Ansible can communicate with the Raspberry Pi.

## 8. Basic Ansible Commands

Check hostname:

```bash
ansible raspberrypi -i inventory.ini -a "hostname"
```

Check disk:

```bash
ansible raspberrypi -i inventory.ini -a "df -h"
```

Check memory:

```bash
ansible raspberrypi -i inventory.ini -a "free -h"
```

## 9. Deploy an Application

Ansible can clone an application from GitHub and configure it on the Raspberry Pi.

Example deployment flow:

```text
GitHub
   |
   v
Ubuntu VM
   |
   | Ansible / SSH
   v
Raspberry Pi
   |
   +-- Clone application
   +-- Install dependencies
   +-- Configure application
   +-- Configure systemd
   +-- Start application
```

A deployment playbook can contain tasks such as:

```yaml
- name: Deploy application
  hosts: raspberrypi
  become: true

  tasks:

    - name: Clone application
      become: false
      ansible.builtin.git:
        repo: "https://github.com/example/project.git"
        dest: "/home/dreamvu/app"
        version: main

    - name: Install required packages
      ansible.builtin.apt:
        name:
          - git
          - python3
        state: present
        update_cache: true
```

Run the playbook:

```bash
ansible-playbook -i inventory.ini deploy.yml
```

## 10. Python Application Deployment

For Python applications, a virtual environment can be created on the Raspberry Pi.

In our Raspberry Pi application setup, `uv` was used for Python dependency management.

Install `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Add it to PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Create a virtual environment:

```bash
uv venv --system-site-packages
```

Install project dependencies:

```bash
uv sync
```

`--system-site-packages` is useful when an application needs Python libraries provided by the Raspberry Pi operating system.

## 11. systemd Service

For applications that should run continuously, systemd can be used.

Example:

```ini
[Unit]
Description=Application Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=dreamvu
WorkingDirectory=/home/dreamvu/app
ExecStart=/home/dreamvu/app/.venv/bin/python main.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

After creating or modifying a service:

```bash
sudo systemctl daemon-reload
```

Start:

```bash
sudo systemctl start application.service
```

Enable at boot:

```bash
sudo systemctl enable application.service
```

Check status:

```bash
sudo systemctl status application.service
```

View logs:

```bash
sudo journalctl -u application.service -f
```

## 12. Managing Multiple Raspberry Pis

The main advantage of Ansible is that the same playbook can manage multiple devices.

Example inventory:

```ini
[raspberrypi]
pi1 ansible_host=192.168.1.17
pi2 ansible_host=192.168.1.18
pi3 ansible_host=192.168.1.19

[raspberrypi:vars]
ansible_user=dreamvu
```

Test all devices:

```bash
ansible raspberrypi -i inventory.ini -m ping
```

Deploy to all:

```bash
ansible-playbook -i inventory.ini deploy.yml
```

Deploy to only one device:

```bash
ansible-playbook -i inventory.ini deploy.yml --limit pi1
```

## 13. Useful Ansible Commands

Syntax check:

```bash
ansible-playbook -i inventory.ini deploy.yml --syntax-check
```

Dry run:

```bash
ansible-playbook -i inventory.ini deploy.yml --check
```

Show inventory:

```bash
ansible-inventory -i inventory.ini --list
```

Run with detailed output:

```bash
ansible-playbook -i inventory.ini deploy.yml -vvv
```

## 14. Troubleshooting

### SSH does not work

```bash
ssh dreamvu@192.168.1.17
```

Check the Raspberry Pi IP:

```bash
hostname -I
```

Check SSH service on Raspberry Pi:

```bash
sudo systemctl status ssh
```

### Ansible ping fails

```bash
ansible raspberrypi -i inventory.ini -m ping -vvv
```

Check that:

- The Raspberry Pi is powered on.
- Both machines are on the same network.
- The IP address is correct.
- SSH works manually.
- The inventory username is correct.

### GitHub cannot be reached

Test Internet access:

```bash
ping -c 4 google.com
```

Test GitHub DNS:

```bash
getent hosts github.com
```

## 15. Security

Do **not** commit private credentials to GitHub.

Never upload:

```text
*.pem
*.key
SSH private keys
passwords
GitHub tokens
Ansible Vault passwords
```

Example `.gitignore`:

```gitignore
*.pem
*.key
.vault_pass
*.retry
.venv/
__pycache__/
*.pyc
```

Keep real infrastructure details and secrets out of public repositories when possible.

## 16. GitHub Workflow

The Ansible project can be maintained using Git:

```bash
cd ~/ansible-raspberrypi

git status
git add .
git commit -m "Update Ansible configuration"
git push
```

Recommended workflow:

```text
Make change
    |
    v
Test Ansible
    |
    v
Test Raspberry Pi
    |
    v
Commit
    |
    v
Push to GitHub
```

## 17. Project Goal

The goal is to replace repeated manual Raspberry Pi configuration with a single automated deployment process.

Instead of manually configuring every Raspberry Pi:

```text
Install
Configure
Copy files
Install dependencies
Configure service
Start application
```

Ansible can perform the process automatically:

```bash
ansible-playbook -i inventory.ini deploy.yml
```

This makes the deployment:

- Repeatable
- Consistent
- Easier to maintain
- Scalable to multiple Raspberry Pis

## 18. Next Steps

Future improvements can include:

- Ansible Roles
- Separate development and production inventories
- Configuration variables
- Automated application updates
- Multiple Raspberry Pi deployment
- Service health checks
- Ansible Vault for secrets
- CI/CD integration

---

## Author

**Shivshankar Kumbar**

Ansible Raspberry Pi Deployment Project

GitHub Repository:

https://github.com/shivshankar003/setupansible
