# WPU Raspberry Pi Ansible Deployment

This project is used to deploy and manage the **WPU application** on one or many Raspberry Pis using **Ansible**.

The goal is simple:

> Keep one WPU project on the Ansible controller (Ubuntu VM), connect Raspberry Pis through SSH, and use one Ansible command to deploy the same application and configuration to all selected Pis.

---

## 1. How the system works

```text
                         Ubuntu VM
                  Ansible Controller
                         |
                         |
                    SSH connection
                         |
        +----------------+----------------+
        |                |                |
      Pi01              Pi02             Pi03 ...
        |                |                |
   WPU application  WPU application  WPU application
```

The local WPU project is stored on the Ubuntu VM:

```text
/media/sf_clean-wpu
```

Ansible copies the project to each Raspberry Pi:

```text
/media/sf_clean-wpu
        |
        | Ansible + rsync
        v
/home/dreamvu/amu-wpu1
```

Ansible also:

- prepares the Raspberry Pi
- installs required Linux packages
- creates the Python virtual environment
- installs the WPU application
- creates WPU configuration
- configures systemd services
- configures the device identity
- installs/configures Grafana Alloy
- starts the selected WPU mode

---

# 2. Main technologies

| Technology | Purpose |
|---|---|
| Ubuntu VM | Ansible control machine |
| Ansible | Automation and deployment |
| SSH | Connection from VM to Raspberry Pis |
| rsync | Copies WPU project files |
| Raspberry Pi | Target device |
| Python virtual environment | Runs WPU Python application |
| systemd | Starts/stops WPU services |
| Grafana Alloy | Collects/forwards logs |
| WPU | Main application |

---

# 3. Project structure

```text
ansible-raspberrypi/
│
├── ansible.cfg
├── README.md
│
├── inventory/
│   └── production/
│       ├── hosts.ini
│       ├── group_vars/
│       │   └── raspberrypi.yml
│       └── host_vars/
│
├── playbooks/
│   ├── common.yml
│   ├── configure-identity.yml
│   ├── wpu-directories.yml
│   ├── python.yml
│   ├── wpu-deploy.yml
│   ├── alloy.yml
│   ├── start-wpu.yml
│   ├── fleet-deploy.yml
│   ├── check-fleet.yml
│   └── check-mode.yml
│
├── roles/
│   ├── common/
│   ├── python/
│   ├── wpu/
│   └── alloy/
│
├── deploy.yml
├── deploy-old.yml
└── wpu.playbook.updated
```

Some files such as `deploy-old.yml` and `wpu.playbook.updated` are older/reference files. The main deployment flow is `playbooks/fleet-deploy.yml`.

---

# 4. Inventory

The inventory tells Ansible **which Raspberry Pis exist and how to connect to them**.

Example:

```ini
[raspberrypi]
pi01 ansible_host=10.47.245.202
pi02 ansible_host=10.47.245.67

[raspberrypi:vars]
ansible_user=dreamvu
ansible_python_interpreter=/usr/bin/python3.11
```

### Adding another Raspberry Pi

For example:

```ini
[raspberrypi]
pi01 ansible_host=10.47.245.202
pi02 ansible_host=10.47.245.67
pi03 ansible_host=10.47.245.80
```

After SSH access is configured, the new Pi can be included in the same deployment.

The important idea is:

```text
Add Pi to inventory
        ↓
Test SSH
        ↓
Run the same deployment command
```

There is no need to create a separate deployment playbook for every Pi.

---

# 5. Configuration variables

Common Raspberry Pi and WPU settings are stored in:

```text
inventory/production/group_vars/raspberrypi.yml
```

This contains settings such as:

```yaml
wpu_user: "dreamvu"
wpu_project_dir: "/home/dreamvu/amu-wpu1"
```

It also contains:

- WPU backend configuration
- WPU runtime modes
- slideshow settings
- face-recognition settings
- logging settings
- Grafana Alloy/Loki configuration

This means configuration can be changed centrally and then deployed to all Pis.

---

# 6. WPU runtime modes

The project supports these WPU service modes:

```yaml
wpu_mode_services:
  server: "slideshow-server.service"
  diagnostic: "slideshow-diagnostic.service"
  only: "slideshow-only.service"
```

The current working configuration uses:

```yaml
wpu:
  mode: "diagnostic"
```

### Modes

| Mode | Service | Purpose |
|---|---|---|
| server | `slideshow-server.service` | Normal WPU server operation |
| diagnostic | `slideshow-diagnostic.service` | Diagnostic/testing mode |
| only | `slideshow-only.service` | Slideshow-only operation |

---

# 7. Playbooks

## `common.yml`

Prepares the Raspberry Pi.

Main responsibilities:

- checks/corrects the Raspberry Pi system clock
- re-enables network time synchronization
- updates the APT package cache
- installs required Linux/Python/GTK/Picamera2/GStreamer dependencies

This is the basic system preparation step.

---

## `configure-identity.yml`

Creates the persistent WPU device identity.

The identity is based on the Ansible inventory hostname.

Example:

```text
pi01 -> /etc/wpu-client/device-id -> pi01
pi02 -> /etc/wpu-client/device-id -> pi02
```

This allows every Raspberry Pi to have a unique WPU identity without maintaining a separate identity file for each Pi in Ansible.

---

## `wpu-directories.yml`

Creates the directories required by the WPU application.

Examples:

```text
/home/dreamvu/amu-wpu1
/home/dreamvu/amu-wpu1/data
/home/dreamvu/amu-wpu1/models
/home/dreamvu/amu-wpu1/config
```

It also prepares the WPU log directory.

---

## `python.yml`

Prepares the WPU Python environment.

It:

1. checks whether `.venv` already exists
2. creates the virtual environment when required
3. upgrades pip
4. installs the WPU project/dependencies
5. verifies that the WPU Python interpreter exists

The virtual environment is created with:

```text
--system-site-packages
```

This allows the WPU environment to use Raspberry Pi system packages such as Picamera2/GTK components.

---

## `wpu-deploy.yml`

Deploys the actual WPU application.

The current deployment source is the local VM directory:

```text
/media/sf_clean-wpu/
```

The destination on every Raspberry Pi is:

```text
/home/dreamvu/amu-wpu1/
```

The deployment uses rsync/synchronize to copy the application.

Files that should not be copied from the controller are excluded, such as:

```text
.git/
.venv/
__pycache__/
.pyc files
pytest cache
```

The project ownership is then set to:

```text
dreamvu:dreamvu
```

---

## `alloy.yml`

Installs and configures **Grafana Alloy**.

Main responsibilities:

- installs Grafana Alloy
- installs/configures the Alloy configuration
- configures log collection
- adds Alloy to the required system group
- enables and starts the Alloy service

The current Loki configuration is intended for the Loki server configured in the project.

---

## `start-wpu.yml`

Starts the selected WPU runtime service.

It reads:

```yaml
wpu.mode
```

and converts that mode into the correct systemd service.

Example:

```text
diagnostic
   ↓
slideshow-diagnostic.service
```

It then verifies that the selected service is active.

The service verification uses retries so Ansible can wait for systemd/application startup instead of checking only once immediately after starting the service.

---

## `fleet-deploy.yml`

This is the **main deployment playbook**.

Instead of manually running every playbook one by one, it combines the complete deployment flow:

```text
common
   ↓
identity
   ↓
directories
   ↓
Python environment
   ↓
WPU application
   ↓
Grafana Alloy
   ↓
start WPU
```

The command used for the complete deployment is:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

This is the main command to remember.

---

## `check-fleet.yml`

Used to check the state of the Raspberry Pi fleet.

It can be used before/after deployment to confirm connectivity and system/application status.

---

## `check-mode.yml`

Used to check the configured WPU runtime mode.

This helps confirm whether a Pi is configured for:

```text
server
diagnostic
only
```

---

# 8. Roles

Ansible roles keep the deployment logic organized.

## `roles/common`

Contains common Raspberry Pi preparation tasks.

```text
roles/common/tasks/main.yml
```

---

## `roles/python`

Contains Python environment setup.

```text
roles/python/tasks/main.yml
```

---

## `roles/wpu`

Contains the main WPU deployment logic.

```text
roles/wpu/
├── tasks/
│   ├── main.yml
│   ├── deploy.yml
│   ├── config.yml
│   ├── systemd.yml
│   └── mode.yml
├── handlers/
└── templates/
```

The templates generate WPU configuration and systemd service files.

---

## `roles/alloy`

Contains Grafana Alloy installation/configuration.

```text
roles/alloy/
├── tasks/main.yml
├── handlers/main.yml
└── templates/config.alloy.j2
```

---

# 9. Complete deployment flow

When the main command is executed:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

the flow is:

```text
1. Read inventory
        ↓
2. Connect to Raspberry Pis through SSH
        ↓
3. Prepare operating system
        ↓
4. Configure device identity
        ↓
5. Create WPU directories
        ↓
6. Create/check Python virtual environment
        ↓
7. Copy WPU application from the VM
        ↓
8. Configure WPU
        ↓
9. Configure systemd services
        ↓
10. Configure Grafana Alloy
        ↓
11. Start selected WPU service
        ↓
12. Verify service status
```

---

# 10. Deploy to one Raspberry Pi

Testing one Pi is useful before deploying to the complete fleet.

For `pi01`:

```bash
ansible-playbook \
  -i inventory/production/hosts.ini \
  playbooks/fleet-deploy.yml \
  --limit pi01
```

For `pi02`:

```bash
ansible-playbook \
  -i inventory/production/hosts.ini \
  playbooks/fleet-deploy.yml \
  --limit pi02
```

---

# 11. Deploy to the complete fleet

Run:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

Every host under:

```ini
[raspberrypi]
```

will be processed.

---

# 12. Test Ansible connection

Before deployment, test that the Pis are reachable:

```bash
ansible raspberrypi \
  -i inventory/production/hosts.ini \
  -m ping
```

Expected result:

```text
pi01 | SUCCESS => ...
pi02 | SUCCESS => ...
```

---

# 13. Syntax check

Before running a playbook:

```bash
ansible-playbook \
  -i inventory/production/hosts.ini \
  playbooks/fleet-deploy.yml \
  --syntax-check
```

If there are no YAML/playbook errors, Ansible reports a successful syntax check.

---

# 14. Check WPU service

Check the service directly on a Raspberry Pi:

```bash
ansible pi01 \
  -i inventory/production/hosts.ini \
  -b \
  -m shell \
  -a 'systemctl status slideshow-diagnostic.service --no-pager -l'
```

Check only whether it is active:

```bash
ansible pi01 \
  -i inventory/production/hosts.ini \
  -b \
  -m shell \
  -a 'systemctl is-active slideshow-diagnostic.service'
```

Expected:

```text
active
```

---

# 15. View WPU logs

For diagnostic mode:

```bash
ansible pi01 \
  -i inventory/production/hosts.ini \
  -b \
  -m shell \
  -a 'journalctl -u slideshow-diagnostic.service -n 80 --no-pager'
```

For the normal server:

```bash
ansible pi01 \
  -i inventory/production/hosts.ini \
  -b \
  -m shell \
  -a 'journalctl -u slideshow-server.service -n 80 --no-pager'
```

---

# 16. Manual service commands on a Raspberry Pi

SSH to a Pi:

```bash
ssh dreamvu@PI_IP_ADDRESS
```

Check services:

```bash
systemctl status slideshow-server.service
systemctl status slideshow-diagnostic.service
systemctl status slideshow-only.service
```

Start diagnostic mode:

```bash
sudo systemctl start slideshow-diagnostic.service
```

Stop diagnostic mode:

```bash
sudo systemctl stop slideshow-diagnostic.service
```

Restart diagnostic mode:

```bash
sudo systemctl restart slideshow-diagnostic.service
```

---

# 17. Important deployment rule

The **source of truth for the WPU application** in this version of the project is the local VM directory:

```text
/media/sf_clean-wpu
```

Do not create a separate WPU copy for every Raspberry Pi on the Ansible controller.

Instead:

```text
One WPU source
      ↓
Ansible
      ↓
Many Raspberry Pis
```

This makes updates easier.

---

# 18. Updating the WPU application

When the WPU source on the VM changes:

```text
/media/sf_clean-wpu
```

run the same deployment command:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

Ansible synchronizes the changed files to the Raspberry Pis.

The same process can be used for:

- code changes
- configuration changes
- systemd changes
- dependency changes
- new Raspberry Pis

---

# 19. Scaling to many Raspberry Pis

The design is intended to scale.

For example:

```ini
[raspberrypi]
pi01 ansible_host=10.47.245.202
pi02 ansible_host=10.47.245.67
pi03 ansible_host=10.47.245.80
pi04 ansible_host=10.47.245.81
...
```

The Ansible configuration uses:

```ini
forks = 50
```

so the controller is configured to handle parallel execution for a larger fleet.

The operational model stays the same:

```text
Add device to inventory
        ↓
Ensure SSH access
        ↓
Test ping
        ↓
Run fleet-deploy.yml
```

---

# 20. Recommended normal workflow

### First time setup

```bash
cd ~/ansible-raspberrypi
```

Test connection:

```bash
ansible raspberrypi -i inventory/production/hosts.ini -m ping
```

Check syntax:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml --syntax-check
```

Deploy:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

### Later updates

Usually only these are needed:

```bash
cd ~/ansible-raspberrypi

ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

---

# 21. Troubleshooting

## SSH connection fails

Test:

```bash
ansible raspberrypi -i inventory/production/hosts.ini -m ping
```

Also test direct SSH:

```bash
ssh dreamvu@PI_IP_ADDRESS
```

---

## WPU service is not active

Check:

```bash
sudo systemctl status slideshow-diagnostic.service --no-pager -l
```

Then:

```bash
sudo journalctl -u slideshow-diagnostic.service -n 80 --no-pager
```

---

## Check whether the WPU files were copied

On the Raspberry Pi:

```bash
ls -la /home/dreamvu/amu-wpu1
```

---

## Check the Python environment

```bash
ls -la /home/dreamvu/amu-wpu1/.venv/bin/python
```

---

# 22. Current architecture summary

```text
                  GIT / PROJECT REPOSITORY
                           |
                           |
                     Ansible Project
                           |
                    Ubuntu VirtualBox VM
                           |
              +------------+------------+
              |                         |
         Inventory                 WPU Source
              |              /media/sf_clean-wpu
              |                         |
              +------------+------------+
                           |
                         SSH
                           |
             +-------------+-------------+
             |             |             |
            Pi01          Pi02          Pi03 ...
             |             |             |
           WPU           WPU           WPU
             |             |             |
        systemd        systemd        systemd
             |             |             |
          Alloy         Alloy         Alloy
```

## Main command

The main command for deployment is:

```bash
ansible-playbook -i inventory/production/hosts.ini playbooks/fleet-deploy.yml
```

This is the central command used to deploy the WPU application across the Raspberry Pi fleet.
