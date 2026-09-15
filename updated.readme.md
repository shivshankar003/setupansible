# WPU Raspberry Pi Fleet Deployment with Ansible

Automated deployment and management of the WPU Client application across Raspberry Pi devices using **Ansible**.

The deployment architecture is designed to support a single Raspberry Pi initially and scale to multiple devices and eventually a larger fleet.

---

## Project Goal

The objective of this project is to build a repeatable and reliable deployment system for the WPU Client on Raspberry Pi devices.

The system is designed to provide:

- Automated Raspberry Pi provisioning
- Application deployment using Ansible
- Per-device identity management
- Centralized configuration management
- Python virtual environment validation
- systemd service management
- Diagnostic and production runtime modes
- Face-recognition diagnostic enrollment
- Grafana Alloy log collection
- Fleet-ready inventory structure
- Future deployment to multiple Raspberry Pis

---

# Architecture

```text
                   Ubuntu VM
              Ansible Control Node
                       |
              Production Inventory
                       |
        +--------------+--------------+
        |              |              |
       Pi01           Pi02           Pi03
        |              |              |
       WPU            WPU            WPU
        |              |              |
        +--------------+--------------+
                       |
                 WPU Backend
                192.168.1.19:8000

Monitoring:

Raspberry Pi
     |
     v
Grafana Alloy
     |
     v
Loki
192.168.1.10:3100
```

---

# Current Deployment Status

| Phase | Status |
|---|---|
| Phase 1 — Ansible Foundation | ✅ Complete |
| Phase 2 — Device Identity | ✅ Complete |
| Phase 3.1 — System Dependencies | ✅ Complete |
| Phase 3.2 — WPU Directories | ✅ Complete |
| Phase 3.3 — Python Environment | ✅ Complete |
| Phase 3.4 — Application Deployment | ✅ Complete |
| Phase 3.5 — Asset Deployment | ⚠️ Partial |
| Phase 3.6 — Configuration Management | ✅ Complete |
| Phase 3.7 — systemd Management | ✅ Complete |
| Phase 3.8 — Runtime Mode Management | ✅ Complete |
| Phase 3.9 — Diagnostic Health Validation | ✅ Complete |
| Phase 3.10 — Diagnostic Runtime Test | ✅ Complete |
| Alloy Deployment | ✅ Complete |
| Loki Connectivity | ⏳ Pending |
| WPU Backend Connectivity | ⏳ Pending |
| Production Runtime Validation | ⏳ Pending |
| Multi-Pi Deployment | ⏳ Pending |
| Fleet Deployment | ⏳ Pending |

---

# Phase 1 — Ansible Foundation

## Objective

Create the Ansible control environment and establish reliable communication between the Ubuntu VM and Raspberry Pi.

### Control machine

```text
Ubuntu VirtualBox VM
```

Project:

```text
~/ansible-raspberrypi
```

Ansible version used:

```text
ansible-core 2.20.1
```

### Inventory

```text
inventory/production/hosts.ini
```

Current Pi:

```ini
[raspberrypi]
pi01 ansible_host=192.168.1.17

[raspberrypi:vars]
ansible_user=dreamvu
ansible_python_interpreter=/usr/bin/python3.11
```

### Ansible configuration

```ini
[defaults]
roles_path = ./roles
host_key_checking = False
interpreter_python = auto_silent
```

### Connectivity validation

```bash
ansible pi01 -i inventory/production/hosts.ini -m ping
```

Result:

```text
SUCCESS / pong
```

Sudo validation:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m command -a whoami
```

Result:

```text
root
```

### Phase 1 status

✅ Complete

---

# Phase 2 — Device Identity

## Objective

Give every Raspberry Pi a persistent identity so the fleet can distinguish devices.

Device identity was configured using:

```text
/etc/wpu-client/device-id
```

For Pi01:

```text
pi01
```

Identity deployment playbook:

```text
playbooks/configure-identity.yml
```

Run:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/configure-identity.yml
```

Verify:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'cat /etc/wpu-client/device-id'
```

Result:

```text
pi01
```

The WPU application reports:

```text
pi01 (from /etc/wpu-client/device-id)
```

### Phase 2 status

✅ Complete

---

# Phase 3 — WPU Deployment

## Phase 3.1 — System Dependencies

Installed required Raspberry Pi dependencies using Ansible.

Playbook:

```text
playbooks/common.yml
```

Run:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/common.yml
```

Important dependencies include:

```text
python3
python3-venv
python3-dev
python3-picamera2
python3-gi
gir1.2-gtk-4.0
libgtk-4-1
libgl1
libglib2.0-0
libcap-dev
gstreamer1.0-tools
gstreamer1.0-plugins-base
gstreamer1.0-plugins-good
gstreamer1.0-plugins-bad
gstreamer1.0-plugins-ugly
```

Validated:

```text
Picamera2 import
GTK4 import
Camera availability
```

### Status

✅ Complete

---

# Phase 3.2 — WPU Directory Structure

Created the required application directories.

Playbook:

```text
playbooks/wpu-directories.yml
```

Run:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-directories.yml
```

Created:

```text
/home/dreamvu/amu-wpu1/data
/home/dreamvu/amu-wpu1/data/embeddings
/home/dreamvu/amu-wpu1/data/people
/home/dreamvu/amu-wpu1/data/stock_images
/home/dreamvu/amu-wpu1/models
/home/dreamvu/amu-wpu1/config
/var/log/wpu-client
```

### Idempotency

The playbook was re-run successfully without unnecessary changes.

### Status

✅ Complete

---

# Phase 3.3 — Python Environment

Validated the WPU virtual environment:

```text
/home/dreamvu/amu-wpu1/.venv
```

Python interpreter:

```text
/usr/bin/python3.11
```

Important validated packages include:

```text
numpy 1.26.4
opencv 4.11.0
PyYAML
PyGObject
Picamera2
onnxruntime
scipy
httpx
pydantic
psutil
```

Python validation playbook:

```text
playbooks/python.yml
```

Run:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/python.yml
```

### Status

✅ Complete

---

# Phase 3.4 — WPU Application Deployment

The application is deployed from:

```text
/media/sf_clean-wpu/
```

to:

```text
/home/dreamvu/amu-wpu1/
```

Deployment uses Ansible synchronization.

Important deployment behavior:

```text
delete: false
```

This protects runtime data from being deleted during application updates.

Protected runtime areas include:

```text
data/embeddings/
data/people/
data/base_assets/
config/config.yaml
```

Application deployment is handled by:

```text
roles/wpu/tasks/deploy.yml
```

Main playbook:

```text
playbooks/wpu-deploy.yml
```

Syntax check:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml --syntax-check
```

Dry run:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml --check
```

Actual deployment:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml
```

### Status

✅ Complete

---

# Phase 3.5 — Assets

The WPU project contains an asset distribution system:

```text
scripts/fetch_assets.sh
scripts/make_assets.sh
```

Asset verification:

```bash
ansible pi01 -i inventory/production/hosts.ini -m shell -a \
'cd /home/dreamvu/amu-wpu1 && ./scripts/fetch_assets.sh --verify'
```

The deployment architecture protects:

```text
embeddings
people
base assets
stock images
```

### Current limitation

A complete production asset source containing all required production embeddings/people/base assets was not available on the control VM during this phase.

The Pi therefore still reports some missing production assets.

### Status

⚠️ Partial

---

# Phase 3.6 — Configuration Management

WPU configuration is managed by Ansible instead of manually editing the Pi.

Template:

```text
roles/wpu/templates/config.yaml.j2
```

Destination:

```text
/home/dreamvu/amu-wpu1/config/config.yaml
```

Important configuration values include:

```yaml
log_level: "INFO"

slideshow:
  enabled: true
  full_screen: true
  advance_time: 2
  scale_mode: "fill"
  sort_mode: "alphabetical"

face_recognition:
  enabled: true
  camera_id: 0
  n: 4
  model: "mobilenet"
  detection_interval: 1
  min_face_size: 100
  display_result: true
  person_timeout: 3
  diagnostic_mode: false
```

Backend configuration is generated from:

```yaml
wpu_server:
  host: "192.168.1.19"
  port: 8000
```

Resulting WPU API endpoints use:

```text
192.168.1.19:8000
```

### Status

✅ Complete

---

# Phase 3.7 — systemd Management

Three WPU systemd services are managed:

```text
slideshow-server.service
slideshow-diagnostic.service
slideshow-only.service
```

Templates:

```text
roles/wpu/templates/slideshow-server.service.j2
roles/wpu/templates/slideshow-diagnostic.service.j2
roles/wpu/templates/slideshow-only.service.j2
```

Systemd configuration:

```text
roles/wpu/tasks/systemd.yml
```

The services are designed to be mutually exclusive.

Runtime behavior:

```text
server      → slideshow-server.service
diagnostic  → slideshow-diagnostic.service
only        → slideshow-only.service
```

### Status

✅ Complete

---

# Phase 3.8 — Runtime Mode Management

Ansible was extended so that the selected WPU mode controls the actual systemd service state.

Mode file:

```text
roles/wpu/tasks/mode.yml
```

Supported modes:

```text
server
diagnostic
only
```

The role:

1. Validates the selected mode.
2. Determines the correct service.
3. Stops non-selected WPU services.
4. Starts the selected service.
5. Enables production server mode for boot persistence.

Example:

```yaml
wpu:
  mode: "diagnostic"
  version: "0.1.0"
```

### Verified diagnostic state

```text
slideshow-server.service      inactive
slideshow-diagnostic.service  active
slideshow-only.service        inactive
```

### Status

✅ Complete

---

# Phase 3.9 — Diagnostic Health Validation

WPU provides an application preflight check:

```bash
.venv/bin/python main.py --check --json
```

Run remotely:

```bash
ansible pi01 -i inventory/production/hosts.ini -m shell -a \
'cd /home/dreamvu/amu-wpu1 && \
.venv/bin/python main.py --check --json'
```

The health check validated:

```text
Dependencies          OK
System                OK
Models                OK
Configuration         OK
Camera                OK
Identity              OK
Logs                  OK
Disk                  OK
Gallery               OK/WARN
```

After enrolling a test person, the diagnostic health result became:

```json
{
  "status": "ok",
  "failed": []
}
```

Warnings remaining were non-blocking:

```text
scenes
gallery
alloy
```

### Status

✅ Complete for diagnostic provisioning

---

# Phase 3.10 — Diagnostic Runtime Test

The diagnostic service was started and successfully remained running.

Verification:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'systemctl status slideshow-diagnostic.service --no-pager -l'
```

Result:

```text
Active: active (running)
```

The WPU application successfully initialized:

```text
camera
libcamera
face recognition
slideshow
display
```

Observed runtime events included:

```text
Displaying overlay
Not recognized
Person timeout
Person left
```

This confirmed that the diagnostic WPU runtime is actually functioning on Pi01.

### Status

✅ Complete

---

# Diagnostic Face Enrollment

A real test face was placed on the Raspberry Pi:

```text
/home/dreamvu/test-face.jpeg
```

Face enrollment command:

```bash
ansible pi01 -i inventory/production/hosts.ini -m shell -a \
'cd /home/dreamvu/amu-wpu1 && \
.venv/bin/python scripts/seed_face.py \
--name "Test Person" \
--face /home/dreamvu/test-face.jpeg'
```

Successfully generated:

```text
data/embeddings/test_person/
```

with embeddings for:

```text
mobilenet
sface
```

The diagnostic health check then passed with no failed checks.

---

# Grafana Alloy Deployment

Grafana Alloy was deployed separately from the application because it is fleet infrastructure.

Playbook:

```text
playbooks/alloy.yml
```

Role:

```text
roles/alloy/
```

Deploy:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/alloy.yml
```

Alloy configuration:

```text
/etc/alloy/config.alloy
```

Alloy is configured to collect WPU systemd journal logs.

WPU services monitored:

```text
slideshow-server.service
slideshow-diagnostic.service
slideshow-only.service
```

Fleet labels:

```text
application = "wpu-client"
device      = "pi01"
```

Loki endpoint:

```text
http://192.168.1.10:3100/loki/api/v1/push
```

Verify Alloy:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'systemctl is-active alloy && \
systemctl is-enabled alloy'
```

Expected:

```text
active
enabled
```

Inspect Alloy configuration:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'grep -E "url =|device|application" /etc/alloy/config.alloy'
```

Inspect Alloy logs:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'journalctl -u alloy --no-pager -n 50'
```

### Current limitation

Alloy itself is running correctly.

However, Pi01 currently cannot reach:

```text
192.168.1.10:3100
```

because the monitoring server network is not currently connected.

Observed error:

```text
no route to host
```

This is a network/Loki availability issue, not an Alloy installation failure.

### Status

✅ Alloy deployment complete

⏳ Loki connectivity pending

---

# Current WPU Runtime Modes

## Diagnostic Mode

Use when the backend server is unavailable and local face-recognition testing is required.

Configuration:

```yaml
wpu:
  mode: "diagnostic"
  version: "0.1.0"
```

and:

```yaml
diagnostic_mode: true
```

Deploy:

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml
```

Check:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'systemctl is-active slideshow-diagnostic.service || true'
```

Follow logs:

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'journalctl -u slideshow-diagnostic.service -f'
```

---

# Production Server Mode

Production configuration:

```yaml
wpu:
  mode: "server"
  version: "0.1.0"
```

and:

```yaml
diagnostic_mode: false
```

Production mode depends on the WPU backend:

```text
192.168.1.19:8000
```

The production service should only be considered runtime-ready after backend connectivity and production assets are available.

---

# Important Deployment Design Decisions

## Application updates do not delete runtime data

The WPU deployment intentionally uses:

```text
delete: false
```

This prevents application synchronization from deleting:

```text
embeddings
people
base assets
local runtime data
```

## Device identity is separated from application code

Identity is maintained in:

```text
/etc/wpu-client/device-id
```

This allows the same application deployment to be used across multiple Raspberry Pis.

## Alloy is separate from the application role

Alloy is treated as:

```text
fleet infrastructure
```

rather than an application dependency.

This allows monitoring to be deployed independently from WPU application updates.

## Runtime modes are mutually exclusive

Only one of the following should run at a time:

```text
server
diagnostic
only
```

Ansible enforces that state.

---

# Important Verification Commands

## Check inventory

```bash
ansible-inventory -i inventory/production/hosts.ini --host pi01
```

## Test connectivity

```bash
ansible pi01 -i inventory/production/hosts.ini -m ping
```

## Dry-run WPU deployment

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml --check
```

## Validate playbook syntax

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml --syntax-check
```

## Deploy WPU

```bash
ansible-playbook -i inventory/production/hosts.ini \
playbooks/wpu-deploy.yml
```

## Health check

```bash
ansible pi01 -i inventory/production/hosts.ini -m shell -a \
'cd /home/dreamvu/amu-wpu1 && \
.venv/bin/python main.py --check --json'
```

## Check WPU service

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'systemctl status slideshow-diagnostic.service --no-pager -l'
```

## Check Alloy

```bash
ansible pi01 -i inventory/production/hosts.ini -b -m shell -a \
'systemctl is-active alloy && systemctl is-enabled alloy'
```

---

# Current Project State

Pi01 has successfully demonstrated the complete local deployment pipeline:

```text
Ubuntu VM
   |
   | Ansible
   v
Raspberry Pi
   |
   +-- Device identity              ✅
   |
   +-- System dependencies          ✅
   |
   +-- Python environment           ✅
   |
   +-- WPU application              ✅
   |
   +-- Configuration                ✅
   |
   +-- systemd                     ✅
   |
   +-- Diagnostic mode             ✅
   |
   +-- Camera                       ✅
   |
   +-- Face enrollment              ✅
   |
   +-- Diagnostic runtime           ✅
   |
   +-- Grafana Alloy                ✅
   |
   +-- Loki connectivity             ⏳
   |
   +-- Production backend           ⏳
   |
   +-- Complete production assets   ⚠️
```

---

# Next Phases

## Phase 4 — Production Runtime Readiness

Separate provisioning readiness from production runtime readiness.

Planned checks:

```text
Backend reachable
Required production assets available
Camera available
Models available
Configuration valid
All required services healthy
```

Production WPU should only start after the runtime readiness requirements are satisfied.

---

## Phase 5 — Multi-Pi Deployment

Add additional Raspberry Pis to the production inventory:

```text
pi01
pi02
pi03
```

Each Pi will have:

```text
unique device_id
unique IP address
same application configuration baseline
same deployment roles
same monitoring configuration
```

---

## Phase 6 — Fleet Validation

Validate:

```text
2 Pis
   ↓
3 Pis
   ↓
small fleet
   ↓
50+ Pis
```

The objective is to deploy the same infrastructure consistently across all devices.

---

## Phase 7 — Fleet Hardening

Future improvements:

```text
rolling deployments
batch deployment
failure handling
health gates
rollback strategy
monitoring
centralized logging
secrets management
asset distribution
configuration overrides
fleet status reporting
```

---

# Repository Structure

```text
ansible-raspberrypi/
│
├── ansible.cfg
│
├── inventory/
│   └── production/
│       ├── hosts.ini
│       ├── group_vars/
│       │   └── raspberrypi.yml
│       └── host_vars/
│           └── pi01.yml
│
├── playbooks/
│   ├── check-fleet.yml
│   ├── configure-identity.yml
│   ├── common.yml
│   ├── wpu-directories.yml
│   ├── python.yml
│   ├── wpu-deploy.yml
│   ├── check-mode.yml
│   └── alloy.yml
│
└── roles/
    ├── common/
    ├── python/
    ├── wpu/
    └── alloy/
```

---

# Final Goal

The final deployment system will allow an administrator to maintain the WPU Raspberry Pi fleet from a central Ansible controller.

Target workflow:

```text
Update Ansible configuration
        |
        v
Update application / configuration
        |
        v
Run Ansible deployment
        |
        v
Pi fleet receives changes
        |
        v
Validate health
        |
        v
Monitor through Alloy + Loki
```

The current implementation has successfully proven this architecture on **Pi01** in diagnostic mode.

The next major milestone is to complete production runtime readiness and then expand the same deployment to **Pi02 and Pi03** before scaling to the larger Raspberry Pi fleet.