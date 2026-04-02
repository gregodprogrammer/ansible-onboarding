# ansible-onboarding
> Production-Ready Ansible DevOps Workstation Setup — DMI Cohort-2 Assignment 32

![DMI](https://img.shields.io/badge/DMI-Cohort--2-orange?logo=bookstack)
![DevOps](https://img.shields.io/badge/DevOps-Micro--Internship-blue?logo=azure-devops)
![Ansible](https://img.shields.io/badge/Ansible-2.17.14-red?logo=ansible)
![Python](https://img.shields.io/badge/Python-3.10.12-blue?logo=python)
![Azure](https://img.shields.io/badge/Azure-canadacentral-blue?logo=microsoftazure)
![Pre-commit](https://img.shields.io/badge/pre--commit-passing-brightgreen)

---

## Overview

This repository documents the setup of a production-ready Ansible developer workstation following real-world team standards. The control node runs on WSL2 Ubuntu 22.04, and the managed node is an Azure Ubuntu 22.04 VM. Anyone following this guide should be able to replicate the entire setup from scratch.

**Architecture:**
```
WSL2 Ubuntu 22.04 (Control Node)
        |
        |  SSH (id_rsa key)
        |
Azure Ubuntu 22.04 VM — canadacentral (Managed Node)
Standard_B2ats_v2 | 2 vCPUs | 1 GiB RAM
```

---

## Prerequisites

- Windows 11 with WSL2 Ubuntu 22.04 installed
- Azure account (free tier works)
- VS Code installed
- Python 3.10+ available in WSL2
- Git configured

---

## PHASE 1 — Azure CLI Setup

### Step 1 — Install Azure CLI (if not installed)
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az --version
```

### Step 2 — Clear any old expired accounts
```bash
az account clear
az logout
```

### Step 3 — Login with device code (most reliable method)
```bash
az login --use-device-code --tenant <YOUR_TENANT_ID>
```
Open browser → go to **https://microsoft.com/devicelogin** → enter the code shown in terminal → sign in.

### Step 4 — Set the correct subscription
```bash
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
az account show --output table
```

**Expected output:** State = Enabled, IsDefault = True

> **Challenge faced:** Old expired Azure account credentials were cached in the CLI. Fix: ran `az account clear` + `az logout` before logging in fresh with the correct tenant ID.

---

## PHASE 2 — Azure VM Creation

### Step 1 — Check your SSH key exists
```bash
ls ~/.ssh/
```
You need `id_rsa` and `id_rsa.pub`. If not, generate one:
```bash
ssh-keygen -t rsa -b 4096 -C "your@email.com" -f ~/.ssh/id_rsa
```

### Step 2 — Create Resource Group
```bash
az group create \
  --name ansible-onboarding-rg \
  --location canadacentral
```

### Step 3 — Create Ubuntu VM
```bash
az vm create \
  --resource-group ansible-onboarding-rg \
  --name ansible-managed-node \
  --image Ubuntu2204 \
  --size Standard_B2ats_v2 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --location canadacentral \
  --output table
```

> **Deviation from assignment:** Used `Standard_B2ats_v2` instead of `Standard_B1s` — only free-eligible size available in canadacentral. Used existing `id_rsa.pub` instead of generating new `id_ed25519` key.

Note your **PublicIpAddress** from the output.

### Step 4 — Open SSH port
```bash
az vm open-port \
  --resource-group ansible-onboarding-rg \
  --name ansible-managed-node \
  --port 22
```

### Step 5 — Test SSH connection
```bash
ssh -i ~/.ssh/id_rsa azureuser@<YOUR_VM_PUBLIC_IP>
```
Type `yes` at fingerprint prompt. You should land at:
```
azureuser@ansible-managed-node:~$
```
Type `exit` to return to WSL2.

---

## PHASE 3 — SSH Agent Setup

### Step 1 — Start SSH agent and add key
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
ssh-add -l
```

### Step 2 — Make agent auto-start on every WSL2 session
```bash
cat >> ~/.bashrc << 'EOF'

# Auto-start SSH agent
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null 2>&1
  ssh-add ~/.ssh/id_rsa > /dev/null 2>&1
fi
EOF
source ~/.bashrc
```

> **Challenge faced:** `ssh-add -l` returned "Could not open a connection to your authentication agent" — SSH agent was not running. Fix: ran `eval "$(ssh-agent -s)"` first, then added auto-start to `~/.bashrc`.

![ssh-add -l showing RSA key loaded](screenshots/ssh-add-l.png)

---

## PHASE 4 — Project Workspace Setup

### Step 1 — Create project folder and venv
```bash
mkdir -p ~/ansible-onboarding && cd ~/ansible-onboarding
python3 -m venv .venv && source .venv/bin/activate
```
Your prompt changes to: `(.venv) greg@DESKTOP...`

### Step 2 — Upgrade pip and install Ansible tools
```bash
python -m pip install --upgrade pip
python -m pip install ansible ansible-lint yamllint pre-commit
```

### Step 3 — Verify installations
```bash
ansible --version
ansible-lint --version
yamllint --version
```

![ansible --version output](screenshots/ansible-version.png)

### Step 4 — Save dependencies
```bash
python -m pip freeze > requirements.txt
```

---

## PHASE 5 — Create Project Config Files

### Step 1 — Create folder structure
```bash
mkdir -p .vscode inventories roles
```

### Step 2 — Create ansible.cfg
```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory            = inventories/
roles_path           = roles:./.ansible/roles
host_key_checking    = True
retry_files_enabled  = False
interpreter_python   = auto_silent
forks                = 10
timeout              = 30
stdout_callback      = yaml
bin_ansible_callbacks = True

[ssh_connection]
pipelining = True
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s
EOF
```

### Step 3 — Create .editorconfig
```bash
cat > .editorconfig << 'EOF'
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true
EOF
```

### Step 4 — Create .vscode/settings.json
```bash
cat > .vscode/settings.json << 'EOF'
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "ansible.python.interpreterPath": "${workspaceFolder}/.venv/bin/python",
  "ansibleLint.enabled": true,
  "yaml.validate": true,
  "files.trimTrailingWhitespace": true,
  "editor.formatOnSave": true
}
EOF
```

### Step 5 — Create .pre-commit-config.yaml
```bash
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.6.1
    hooks:
      - id: ansible-lint
EOF
```

### Step 6 — Create .ansible-lint (prevent venv scanning)
```bash
cat > .ansible-lint << 'EOF'
exclude_paths:
  - .venv/
  - .git/
  - .cache/
warn_list: []
EOF
```

> **Challenge faced:** First run of `pre-commit run --all-files` showed 20,752 failures — ansible-lint was scanning the entire `.venv/` folder (34,443 files). Fix: created `.ansible-lint` config with `exclude_paths: .venv/`.

### Step 7 — Create .gitignore
```bash
cat > .gitignore << 'EOF'
.venv/
__pycache__/
*.pyc
*.pyo
*.retry
.ansible/
*.log
*.pem
*.key
id_rsa
id_rsa.pub
.cache/
.DS_Store
Thumbs.db
EOF
```

---

## PHASE 6 — Git Setup and Pre-commit Hooks

### Step 1 — Check existing Git config
```bash
git config --list
```

### Step 2 — Initialize repo
```bash
git init
git config --global init.defaultBranch main
```

> **Note:** Git identity was already configured as `gregodprogrammer / gregodprogrammer@gmail.com` — left untouched. Only added `init.defaultBranch main` which was missing.

### Step 3 — Install pre-commit hooks
```bash
pre-commit install
```
Expected:
```
pre-commit installed at .git/hooks/pre-commit
```

> **Challenge faced:** `pre-commit install` failed with "git failed, not a git repository" — `git init` had not been run yet. Fix: run `git init` before `pre-commit install`.

### Step 4 — Run smoke test
```bash
pre-commit run --all-files
```
Expected:
```
yamllint............(no files to check) Skipped
Ansible-lint................................Passed
```

![pre-commit run --all-files passing](screenshots/pre-commit-passed.png)

---

## PHASE 7 — VS Code Extensions

Install these extensions in VS Code:

| Extension | Publisher | Purpose |
|---|---|---|
| Ansible | Red Hat | Playbook syntax, linting, snippets |
| YAML | Red Hat | YAML validation and formatting |
| Python | Microsoft | Interpreter selection, venv support |
| EditorConfig | EditorConfig | Consistent formatting across editors |

---

## PHASE 8 — Verify Final Repo Tree

```bash
find . -not -path './.venv/*' -not -path './.git/*' -not -path './.cache/*' | sort
```

Expected output:
```
.
./.ansible-lint
./.editorconfig
./.gitignore
./.pre-commit-config.yaml
./.vscode
./.vscode/settings.json
./ansible.cfg
./inventories
./requirements.txt
./roles
```

![Final repo tree](screenshots/repo-tree.png)

---

## PHASE 9 — Push to GitHub

### Step 1 — Create repo on GitHub
Go to **https://github.com/new** → name: `ansible-onboarding` → Public → No README → Create

### Step 2 — Push
```bash
cd ~/ansible-onboarding
git add .
git commit -m "feat: production-ready Ansible DevOps workstation setup"
git branch -M main
git remote add origin https://github.com/gregodprogrammer/ansible-onboarding.git
git push -u origin main
```

---

## Verification Checklist

- [ ] `ansible --version` runs without errors
- [ ] `ansible-lint --version` runs without errors
- [ ] `pre-commit run --all-files` passes clean
- [ ] `ssh-add -l` shows RSA key loaded
- [ ] SSH into Azure VM works without password prompt
- [ ] Repo tree shows all required files
- [ ] VS Code extensions installed

---

## Full Challenges & Deviations Log

| Challenge | Fix Applied |
|---|---|
| Old expired Azure account cached in CLI | `az logout` + `az account clear` + fresh `az login --use-device-code` |
| SSH agent not running — "Could not open connection" | `eval "$(ssh-agent -s)"` + added auto-start to `~/.bashrc` |
| `ssh-add -l` run inside Azure VM by mistake | Exited VM first, ran on WSL2 control node |
| `pre-commit install` before `git init` | Ran `git init` first |
| ansible-lint scanned `.venv/` — 20,752 false failures | Added `.ansible-lint` with `exclude_paths: .venv/` |
| ansible-lint first run very slow (2-5 mins) | Expected — first run downloads isolated hook environments |
| Assignment requires `id_ed25519` | Used existing `id_rsa` — avoids key sprawl |
| VM size `Standard_B1s` not available | Used `Standard_B2ats_v2` — only free-eligible size in canadacentral |
| `init.defaultBranch` not set | Added via `git config --global init.defaultBranch main` |

---

## DMI Micro-Internship

This project was completed as part of the **DevOps Micro-Internship (DMI) Cohort-2** program.

| Detail | Info |
|---|---|
| Program | DevOps Micro-Internship (DMI) Cohort-2 |
| Assignment | Assignment 1 — Ansible DevOps Workstation Onboarding |
| Candidate | Greg Odi |
| GitHub | [@gregodprogrammer](https://github.com/gregodprogrammer) |
| LinkedIn | [linkedin.com/in/gregodi](https://linkedin.com/in/gregodi) |
