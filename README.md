# Ansible Server Setup

An Ansible playbook for preparing an Ubuntu server with the tools commonly
needed for application deployment:

- Docker Engine, Docker CLI, Buildx, Compose, and containerd
- Python 3, `pip`, virtual-environment support, and development headers
- Node.js 24 and `npm`
- Git, `curl`, CA certificates, and GnuPG

The playbook supports both `x86_64` and `aarch64` hosts when adding the
Docker and NodeSource APT repositories.

## Project structure

```text
.
├── README.md
└── ansible/
    ├── ansible.cfg       # Ansible defaults and privilege escalation settings
    ├── inventory.ini     # Local-test inventory
    └── site.yml          # Server setup playbook
```

## Requirements

### Control machine

- Ansible 2.15 or newer
- SSH client access to the target server
- An SSH private key accepted by the target server

Install Ansible using your operating system's package manager or a Python
virtual environment. For example:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip ansible
```

### Target server

- Ubuntu with APT and `systemd`
- A user that can connect over SSH and use `sudo`
- A supported architecture: `x86_64` or `aarch64`
- Network access to Ubuntu, Docker, and NodeSource package repositories

The playbook uses privilege escalation (`become: true`) for system changes.

## Configure the inventory

Edit [`ansible/inventory.ini`](ansible/inventory.ini) and replace the example
host, address, and user with the details of your server:

```ini
[servers]
myserver ansible_host=203.0.113.10 ansible_user=ubuntu
```

If SSH requires a non-default key or port, provide those options when running
Ansible:

```bash
ansible-playbook \
  -i ansible/inventory.ini \
  --private-key ~/.ssh/my-server-key \
  -e 'ansible_port=2222' \
  ansible/site.yml
```

Do not commit real IP addresses, usernames, or private key paths to the
repository.

## Run the playbook

From the project root, verify connectivity first:

```bash
ansible -i ansible/inventory.ini servers -m ansible.builtin.ping
```

Apply the server configuration:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

To preview the changes without modifying the server:

```bash
ansible-playbook \
  -i ansible/inventory.ini \
  --check \
  --diff \
  ansible/site.yml
```

The playbook updates the APT cache, configures the Docker and NodeSource
repositories, installs the requested packages, starts Docker, adds the
configured user to the `docker` group, and prints the installed versions.

## Configuration

The main variables are defined in [`ansible/site.yml`](ansible/site.yml):

| Variable | Default | Description |
| --- | --- | --- |
| `node_major` | `24` | Node.js major version installed from NodeSource |
| `docker_user` | `{{ ansible_user }}` | User added to the `docker` group |

Override a variable for a run with `--extra-vars`:

```bash
ansible-playbook \
  -i ansible/inventory.ini \
  -e 'node_major=22' \
  ansible/site.yml
```

## After a successful run

The target user must start a new login session before the updated Docker group
membership is available. Either reconnect over SSH or run:

```bash
newgrp docker
```

Then verify the installed tools:

```bash
docker --version
docker compose version
python3 --version
node --version
npm --version
```

## Security notes

The included [`ansible/ansible.cfg`](ansible/ansible.cfg) disables SSH host-key
checking for convenience and enables automatic privilege escalation. Review
these settings before using the playbook outside a controlled testing
environment. In particular, consider enabling host-key checking and managing
SSH credentials through your CI/CD secret store.

The playbook downloads repository signing keys over HTTPS and stores them in
`/etc/apt/keyrings` before configuring the package repositories.