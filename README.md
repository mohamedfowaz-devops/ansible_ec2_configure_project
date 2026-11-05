# Ansible EC2 Configuration Project

Automated configuration of multiple AWS EC2 instances using Ansible from a centralized control node. Roles prepare the servers, install Docker, render the Flask application configuration and set up Nginx as a reverse proxy, with sensitive values protected by Ansible Vault.

## Highlights

- Agentless configuration management over SSH from a single Ansible control node
- Modular, reusable roles: `common`, `docker`, `flask`, `nginx`
- Variables separated from tasks using `group_vars` (`deploy_user`, `app_directory`)
- Jinja2 templates for the Flask configuration and the Nginx site configuration
- Secrets protected with Ansible Vault; the rendered config file is restricted to mode `0600`
- Handlers reload Nginx only when its configuration changes
- Idempotent playbook that is safe to re-run
- Documented break-fix scenarios covering failures encountered during the build

## Architecture

```
Windows Laptop
      |
      | SSH
      ↓
Ansible Control Node (EC2)
      |
      | Private SSH
      ↓
 ┌───────────┐
 │           │
Web-01      Web-02
 EC2         EC2
```

## Tools & Technologies

| Area | Tools |
|---|---|
| Cloud | AWS EC2 |
| Configuration Management | Ansible (Roles, Handlers, Jinja2, Vault) |
| Containers | Docker |
| Web Server | Nginx |
| OS | Ubuntu / Linux |
| Source Control | Git, GitHub |
| Access | SSH key-based authentication |

## Project Structure

```
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

| Role | What it does |
|---|---|
| `common` | Base packages and system preparation |
| `docker` | Installs `docker.io`, starts and enables the service, and adds `deploy_user` to the `docker` group |
| `flask` | Renders the Flask `config.yml` from a Jinja2 template into `app_directory`, owned by `deploy_user` with mode `0600` |
| `nginx` | Installs Nginx, deploys a templated site config to `sites-available`, enables it with a symlink, reloads Nginx via a handler and enables the service |

## Configuration

### `ansible.cfg`

```ini
[defaults]
inventory = ./inventory.ini
private_key_file = /home/ubuntu/<your-key>.pem
remote_user = ubuntu
host_key_checking = False   # lab convenience only
forks = 5
```

### Inventory (`inventory.ini`)

```ini
[webservers]
web01 ansible_host=172.31.1.42
web02 ansible_host=172.31.14.24
```

### Playbook (`site.yml`)

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

## Usage

Check the inventory:

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

Run the playbook:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Verify Docker and Nginx:

```bash
ansible webservers -m command -a "docker --version"
ansible webservers -m shell -a "systemctl is-active nginx"
```

## Security

- Private keys (`*.pem`, `*.key`) are stored outside the repository and excluded with `.gitignore`
- Sensitive variables are encrypted with Ansible Vault
- The rendered Flask config is restricted to its owner (`0600`)
- `host_key_checking = False` is used for this lab only and should be enabled in production

```
*.pem
*.key
group_vars/all/vault.yml
```

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

## Idempotency

Re-running the playbook does not change servers that are already configured. On a second run, tasks report `ok` instead of `changed`, and Nginx is reloaded only when its configuration actually changes.

```bash
ansible-playbook site.yml --ask-vault-pass
```

## Break-Fix Scenarios

Failures encountered in this project, how they were identified, and how they were resolved.

### 1. SSH connection failed (`UNREACHABLE`)
- **Issue:** `ansible all -m ping` returned `UNREACHABLE! Permission denied (publickey)` or a connection timeout.
- **Root cause:** A wrong key path or user in `ansible.cfg`, key file permissions that were too open, or a security group that did not allow SSH from the control node.
- **Resolution:** Verified `private_key_file` and `remote_user`, set `chmod 400` on the key file, allowed port 22 from the control node's security group, and confirmed connectivity with `ansible all -m ping`.

### 2. Privilege escalation failed
- **Issue:** Package installation tasks failed with `Missing sudo password` or `Permission denied`.
- **Root cause:** `become: true` was missing, or the remote user did not have passwordless sudo.
- **Resolution:** Confirmed `become: true` was set in `site.yml`, verified sudo access for the `ubuntu` user and re-ran the playbook.

### 3. Ansible Vault decryption failed
- **Issue:** The playbook failed with `Decryption failed` or `Attempting to decrypt but no vault secrets found`.
- **Root cause:** The wrong vault password was entered, or the playbook was run without `--ask-vault-pass`.
- **Resolution:** Re-ran the playbook with `--ask-vault-pass` and the correct password, and verified the file with `ansible-vault view`.

### 4. Variable undefined in a role or template
- **Issue:** A task failed with `'deploy_user' is undefined` or `'app_directory' is undefined`.
- **Root cause:** The variable was missing from `group_vars/all/vars.yml`, misspelled, or defined in a file that was not loaded for the host group.
- **Resolution:** Checked the `group_vars/all/` files, corrected the variable name and confirmed the value with `ansible-inventory --host web01`.

### 5. Docker permission denied for the deploy user
- **Issue:** Docker commands failed with `permission denied while trying to connect to the Docker daemon socket`.
- **Root cause:** The user had just been added to the `docker` group, and the new group membership only applies to new login sessions.
- **Resolution:** Verified the group assignment with `id`, reconnected the SSH session so the group took effect, and confirmed access with `docker ps`.

### 6. Nginx failed to start or reload after a configuration change
- **Issue:** `systemctl is-active nginx` returned `failed`, or the *Reload Nginx* handler failed.
- **Root cause:** An error in the templated site configuration, a broken symlink in `sites-enabled`, or a conflict with the default site.
- **Resolution:** Validated the config with `nginx -t`, reviewed the service logs, corrected the Jinja2 template and re-ran the playbook.
  ```bash
  sudo nginx -t
  sudo journalctl -u nginx --no-pager -n 50
  ls -l /etc/nginx/sites-enabled/
  ```

### 7. Nginx returned 502 Bad Gateway
- **Issue:** The site returned `502 Bad Gateway` through Nginx.
- **Root cause:** The upstream application was not running, or the `proxy_pass` port in the Nginx template did not match the application's port.
- **Resolution:** Verified the application was running and listening, confirmed the port in the template matched, and reloaded Nginx.
  ```bash
  sudo ss -tulpn | grep -E ':80|:5000'
  curl -i http://127.0.0.1:<app-port>/
  ```

### 8. Playbook was not idempotent
- **Issue:** A task reported `changed` on every run.
- **Root cause:** A `shell` or `command` task was used where a native module (`apt`, `service`, `template`, `file`) was needed.
- **Resolution:** Replaced the shell task with a native module and confirmed a second run reported `changed=0`.

### 9. YAML syntax error
- **Issue:** The playbook failed with `Syntax Error while loading YAML`.
- **Root cause:** Incorrect indentation or a misplaced colon or quote in a task file.
- **Resolution:** Ran `ansible-playbook site.yml --syntax-check`, fixed the indentation and re-ran the playbook.

## Key Learnings

- Structuring Ansible projects with inventories, roles, handlers and `group_vars`
- Writing Jinja2 templates for dynamic configuration
- Protecting secrets with Ansible Vault and restricting file permissions
- SSH key-based authentication and privilege escalation
- Configuring Docker and Nginx on EC2 with Ansible
- Writing idempotent playbooks
- Diagnosing and fixing failures in configuration management

## Future Improvements

- Add a task or role to run the Flask application (as a Docker container or systemd service)
- Remove the default Nginx site to avoid configuration conflicts
- Enable `host_key_checking` and use `serial` for rolling updates
- Add `ansible-lint` and a CI check on the repository
