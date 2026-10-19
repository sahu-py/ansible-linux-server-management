# Linux Server Configuration Management with Ansible Navigator

A role-based Ansible project for configuring  **Ubuntu**
using Ansible Navigator, Ansible Builder, and Podman.

## Goals

-   Install baseline packages using distribution-specific package names.
-   Set the system timezone.
-   Ensure an operations group exists.
-   Enable and start Chrony.
-   Optionally deploy an SSH configuration drop-in with validation
    before reload.

## Architecture


Git project
  -> Ansible Navigator
  -> Execution Environment (Podman)
       Ansible Core + Runner + dependencies



The Execution Environment uses an RPM-based UBI 9 image. Managed servers
can still run Ubuntu.

## Directory structure
``` text
linux-server-configuration-management/
├── README.md
├── .gitignore
├── ansible.cfg
├── ansible-navigator.yml
├── execution-environment.yml
├── requirements.yml
├── inventories/dev/
│   ├── hosts.yml
│   
├── playbooks/
│   └── site.yml
├── roles/linux_baseline/
    ├── vars/main.yml
    ├── tasks/main.yml
    ├── handlers/main.yml
    └── templates/ssh-baseline.conf.j2

```
## Prerequisites

Control node: Ubuntu, Python 3, Podman, Git, OpenSSH client, and
Ansible Development Tools.

Managed nodes: RHEL 9 or Ubuntu, reachable SSH, an automation
account, a configured SSH public key, working repositories, systemd, and
approved sudo/become permissions.

## 1. Install tools

``` bash
sudo apt update
sudo apt install -y podman python3 python3-venv python3-pip git openssh-client

python3 -m venv ~/ansible-dev-venv
source ~/ansible-dev-venv/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible-dev-tools

ansible-navigator --version
ansible-builder --version
ansible-lint --version
podman info
```

Use a Python version supported by your installed Ansible Development
Tools release. Avoid forcing pip to modify system Python.

Activate the virtual environment in every new shell:

``` bash
source ~/ansible-dev-venv/bin/activate
```

## 2. Configure inventory

Edit `inventories/dev/hosts.yml`. Example:

``` yaml
---
all:
  children:
    lab:
      hosts:
        linux-lab-01:
          ansible_host: 192.168.35.134
          ansible_user: demouser
      vars:
        ansible_become: true
```

Replace the documentation-only IP addresses with real host addresses.

## 3. Configure SSH

Install the public key for the automation user on each host, for
example:

``` bash
ssh-copy-id demouser@YOUR_UBUNTU_IP
ssh demouser@YOUR_UBUNTU_IP
```

Keep host-key checking enabled. Verify each host fingerprint through a
trusted channel. The automation user must have approved sudo/become
permissions.

Navigator's EE must access the SSH key and known-hosts data. For a
simple lab, add this option to Navigator commands:

``` bash
--eev "$HOME/.ssh:/root/.ssh:ro"
```



## 4. Build the Execution Environment

Run from the project root:

``` bash
ansible-builder build \
  --file execution-environment.yml \
  --tag localhost/linux-baseline-ee:1.1 \
  --container-runtime podman

podman images
```

The image tag must match `ansible-navigator.yml`. If Builder options
differ in your installed release, run `ansible-builder build --help`.



## 5. Validate the project

``` bash
# Inventory
ansible-navigator inventory -i inventories/dev/hosts.yml --graph --mode stdout

# Syntax
ansible-navigator run playbooks/site.yml \
  -i inventories/dev/hosts.yml --syntax-check --mode stdout

# Lint
ansible-navigator lint playbooks/site.yml --mode stdout

# SSH connectivity and become
ansible-navigator run playbooks/connectivity.yml \
  -i inventories/dev/hosts.yml \
  --eev "$HOME/.ssh:/root/.ssh:ro" --mode stdout
```


## 6. Preview changes

``` bash
ansible-navigator run playbooks/site.yml \
  -i inventories/dev/hosts.yml \
  --eev "$HOME/.ssh:/root/.ssh:ro" \
  --check --diff --mode stdout
```

Check mode is a preview, not a guarantee. Some tasks cannot predict
their full effects without running on the host. Review the output before
applying changes.

## 7. Apply to one test host

``` bash
ansible-navigator run playbooks/site.yml \
  -i inventories/dev/hosts.yml \
  --eev "$HOME/.ssh:/root/.ssh:ro" 
```

## 9. Optional SSH hardening

SSH hardening is disabled by default. Before enabling it, confirm
key-based access and recovery access, keep an existing SSH session open,
and review `/etc/ssh/sshd_config` plus `/etc/ssh/sshd_config.d/`.

To enable the optional drop-in for a test host, set this in
`inventories/dev/group_vars/linux_servers.yml`:

``` yaml
baseline_manage_sshd: true
```
