# Ansible Galaxy Role — `<role_name>`

> **This README applies to all self-hosted Ansible Galaxy roles in the `Infra` org:**
> - `reusable-ansible-galaxy-role-curl`
> - `reusable-ansible-galaxy-role-vim`
> - `reusable-ansible-galaxy-role-htop`
> - `reusable-ansible-galaxy-role-docker`
>
> Replace `<role_name>` and role-specific variables with the values from the table in [Role Reference](#role-reference).

---

## Table of Contents

- [Overview](#overview)
- [Role Reference](#role-reference)
- [Requirements](#requirements)
- [Role Variables](#role-variables)
- [How to Install](#how-to-install)
- [How to Use in a Playbook](#how-to-use-in-a-playbook)
- [Overriding Variables](#overriding-variables)
- [Idempotency](#idempotency)
- [Repository Structure](#repository-structure)
- [Testing](#testing)
- [Design Notes](#design-notes)

---

## Overview

Each role in this collection installs a single package on Ubuntu systems using only `ansible.builtin` modules — no third-party collections required. Roles are:

- **Single-purpose** — one role, one package concern
- **Idempotent** — safe to run multiple times
- **Configurable** — package name and state exposed as variables with sensible defaults
- **Self-contained** — no inter-role dependencies

They are hosted as individual Gitea repos under the `Infra` org and consumed via `ansible-galaxy role install` using the `requirements.yml` pattern.

---

## Role Reference

| Repo | Role name | Package | Default variable | State variable |
|---|---|---|---|---|
| `reusable-ansible-galaxy-role-curl` | `curl` | `curl` | `curl_package` | `curl_state` |
| `reusable-ansible-galaxy-role-vim` | `vim` | `vim` | `vim_package` | `vim_state` |
| `reusable-ansible-galaxy-role-htop` | `htop` | `htop` | `htop_package` | `htop_state` |
| `reusable-ansible-galaxy-role-docker` | `docker` | Docker CE | `docker_packages` | `docker_state` |

The `docker` role differs slightly — it installs Docker CE from the official upstream apt repo (not the Ubuntu default package), adds the GPG key, and enables the `docker` systemd service. All other roles use a single `apt` task.

---

## Requirements

- Ansible ≥ 2.12
- Ubuntu 20.04 (focal), 22.04 (jammy), or 24.04 (noble)
- `become: true` must be set at the play or task level (all roles require root)
- No external collections required — only `ansible.builtin`

---

## Role Variables

### curl / vim / htop

Each of these roles exposes two variables:

| Variable | Default | Description |
|---|---|---|
| `<role>_package` | `<role>` | apt package name to install |
| `<role>_state` | `present` | `present` to install, `absent` to remove |

Examples:

| Role | Package variable | State variable |
|---|---|---|
| curl | `curl_package: curl` | `curl_state: present` |
| vim | `vim_package: vim` | `vim_state: present` |
| htop | `htop_package: htop` | `htop_state: present` |

### docker

The docker role manages the full Docker CE installation from `download.docker.com`. It does not expose a simple package variable — the package list is fixed to the canonical set:

| Variable | Default | Description |
|---|---|---|
| `docker_state` | `present` | `present` to install, `absent` to remove |
| `docker_service_enabled` | `true` | Whether to enable docker at boot |
| `docker_service_state` | `started` | systemd service state (`started`, `stopped`) |

The role installs: `docker-ce`, `docker-ce-cli`, `containerd.io`. The upstream GPG key and apt repo are added automatically.

---

## How to Install

### Via `ansible/requirements.yml` (recommended)

Add the roles you need to your `ansible/requirements.yml`:

```yaml
roles:
  - name: curl
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-curl.git
    scm: git
    version: main

  - name: vim
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-vim.git
    scm: git
    version: main

  - name: htop
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-htop.git
    scm: git
    version: main

  - name: docker
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-docker.git
    scm: git
    version: main
```

Then install all roles before running the playbook:

```bash
ansible-galaxy role install --force -r ansible/requirements.yml
```

The `--force` flag ensures the latest version is always pulled when `version: main` is used. For production stability, pin `version:` to a git tag (e.g. `v1.0.0`) and drop `--force`.

### Direct install (single role)

```bash
ansible-galaxy role install \
  git+http://gitea.local/infra/reusable-ansible-galaxy-role-curl.git,main,curl
```

---

## How to Use in a Playbook

Include roles in `ansible/playbook.yml` like any other role:

```yaml
---
- name: Configure VM
  hosts: all
  become: true
  gather_facts: true

  roles:
    - role: common     # local role
      tags: [common]

    - role: docker     # local or Galaxy role
      tags: [docker]

    - role: curl       # Galaxy role (installed via requirements.yml)
      tags: [curl]

    - role: vim
      tags: [vim]

    - role: htop
      tags: [htop]
```

Run only specific roles using tags:

```bash
ansible-playbook -i inventory.ini ansible/playbook.yml --tags curl,vim
```

---

## Overriding Variables

Pass variable overrides in the `roles:` block or via `--extra-vars`:

**In the playbook:**
```yaml
roles:
  - role: curl
    vars:
      curl_state: absent   # uninstall curl
```

**Via command line:**
```bash
ansible-playbook -i inventory.ini ansible/playbook.yml \
  --extra-vars "vim_state=absent"
```

**docker role — disable service autostart:**
```yaml
roles:
  - role: docker
    vars:
      docker_service_enabled: false
      docker_service_state: stopped
```

---

## Idempotency

All roles are fully idempotent:

- `curl`, `vim`, `htop` — the `apt` module's `state: present` is a no-op if the package is already installed
- `docker` — the GPG key conversion step uses `args: creates:` to skip if the `.gpg` file already exists; the `apt_repository` and `systemd` tasks are natively idempotent

Safe to run the playbook multiple times without side effects.

---

## Repository Structure

Each role repo follows the standard Ansible Galaxy layout:

```
reusable-ansible-galaxy-role-<name>/
├── tasks/
│   └── main.yml          # core task(s)
├── defaults/
│   └── main.yml          # default variable values
├── meta/
│   └── main.yml          # Galaxy metadata (author, platforms, tags)
├── tests/
│   ├── inventory         # localhost ansible_connection=local
│   └── test.yml          # minimal playbook for local testing
└── README.md             # this file
```

The `meta/main.yml` declares compatibility with Ubuntu focal, jammy, and noble, and sets `dependencies: []` — no role depends on another.

---

## Testing

Each role includes a minimal test playbook under `tests/`:

```bash
# Install the role locally for testing
ansible-galaxy role install \
  git+http://gitea.local/infra/reusable-ansible-galaxy-role-curl.git,main,curl

# Run the test playbook (installs to localhost)
ansible-playbook -i tests/inventory tests/test.yml --become
```

The test playbook applies the role to `localhost` using a local connection — no remote VM needed. This is the same flow used by the `ansible-check` reusable workflow's syntax-check step.

---

## Design Notes

**Why one repo per role?**
Each role is independently versioned, independently tested, and independently consumable. A single monolithic Galaxy repo would couple unrelated packages — if `docker` changes, you'd bump a version that also affects `curl` and `vim` consumers.

**Why `version: main` as default?**
During homelab development, `main` always reflects the latest working state. When these roles are used in production workloads, pin to a semver tag for reproducibility. Add a CI workflow to the role repo that tags on merge to `main`.

**Why not use Ansible Collections?**
Collections are appropriate when packaging multiple roles, modules, and plugins together with a shared namespace. For single-package utility roles, the simpler Galaxy role format (no collection wrapper, no `galaxy.yml`) is sufficient and easier to maintain.

**Why self-host on Gitea instead of Ansible Galaxy (galaxy.ansible.com)?**
The homelab runs entirely on-premises. Sourcing roles from `gitea.local` keeps the dependency graph internal — no internet access required during CI, and the versions are under full control. The `scm: git` source type in `requirements.yml` works identically to public Galaxy for `ansible-galaxy role install`.

---

## Infrastructure Created and Maintained by

**Ali Ahmed**  
Building infrastructure, automation, and DevOps workflows

**Contact**

[![GitHub](https://img.shields.io/badge/GitHub-%20ali%20ahmed-black?style=for-the-badge&logo=github)](https://github.com/jeffreyalie)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%20ali%20ahmed-blue?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/ali-ahmed-261755252/)
