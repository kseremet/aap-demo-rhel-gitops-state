![Status](https://img.shields.io/badge/Status-Ready-green)
![Red Hat AAP](https://img.shields.io/badge/AAP-2.6%2B-red)
![RHEL System Roles](https://img.shields.io/badge/RHEL%20System%20Roles-certified-red)

# RHEL GitOps Desired State

## Introduction

This repository is the human-maintained GitOps state for **From Admin to
Maintainer: Agentic Ops and GitOps for RHEL**. It shows Linux administrators how
to stop mutating servers with manual commands and instead declare RHEL state in
Git for AAP to reconcile.

The repository intentionally contains no VM provisioning or generated
connection data. AAP inventory supplies the visible host management addresses;
this repository supplies the configuration intent.

## Prerequisites

| Requirement | Details |
|---|---|
| Platform repository | `aap-demo-rhel-gitops-platform` bootstrapped and MACs populated |
| AAP | Job Template `JT - RHEL GitOps Reconciliation` configured by the platform CasC |
| Target hosts | 4 RHEL 9 VMs: web-01, web-02, db-01, app-01 |
| Ansible collections | `redhat.rhel_system_roles` |

## Quick Start

### 1. Clone and install collections

```bash
git clone https://github.com/kseremet/aap-demo-rhel-gitops-state.git
cd aap-demo-rhel-gitops-state

cp ansible.cfg.example ansible.cfg
ansible-galaxy collection install \
  -r collections/requirements.yml \
  -p collections
```

### 2. Pre-demo: populate host MACs

Run this from the **platform repository** before the event. It replaces
`CHANGE_ME_*_MAC` placeholders with the actual MAC addresses discovered from
the VMs:

```bash
cd ../aap-demo-rhel-gitops-platform
source .venv/bin/activate

ansible-playbook utils/populate_host_vars.yml -i environments/lab/inventory.yml
```

Commit the result in the state repository:

```bash
cd ../aap-demo-rhel-gitops-state
git add host_vars && git commit -m "Populate host MAC addresses" && git push
```

### 3. Local reconciliation (optional)

Use local runs to validate syntax and behavior quickly before the event:

```bash
ansible-playbook \
  -i ../aap-demo-rhel-gitops-platform/environments/lab/inventory.yml \
  site.yml
```

Use AAP for the audience-facing workflow: project sync, job launch, output, and
repeatable reconciliation after each Git commit.

## Live Demo Script

### Phase 1 — Fleet baseline

Every host receives the same baseline: subscription, repositories,
time synchronization, SSH hardening, sudo policy, firewall, storage, and kernel
settings. This is the initial state — no group or host specialization yet.

```bash
# Already committed. Launch the AAP reconciliation job.
```

Explain: every RHEL System Role applied its declared state. No manual commands
were needed. The Git files are the source of truth.

### Phase 2 — Host-specific networking

Uncomment the static internal interface sections. Each host gets a unique MAC
and IP defined in its `host_vars/<host>/network.yml`:

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh network

git diff
git add host_vars
git commit -m "Declare static internal network interfaces"
git push
```

Launch the reconciliation job. Explain: the commit is the change event; the
System Role is the reconciler. No administrator needs to run `nmcli`.

### Phase 3 — Web server storage

Uncomment the web content volume and its disk binding:

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh storage-web

git diff
git add group_vars/web_servers host_vars/web-01 host_vars/web-02
git commit -m "Declare web server storage"
git push
```

Launch reconciliation. Verify `/var/www` exists only on web-01 and web-02.

### Phase 4 — Database server storage

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh storage-db

git add group_vars/database_servers host_vars/db-01
git commit -m "Declare database server storage"
git push
```

Verify `/var/lib/pgsql` and WAL volumes exist only on db-01.

### Phase 5 — Application server storage

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh storage-app

git add group_vars/app_servers host_vars/app-01
git commit -m "Declare application server storage"
git push
```

Verify `/opt/app` exists only on app-01.

### Phase 6 — Group kernel tuning

Uncomment the kernel tuning overlays for each server role:

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh kernel-web
git add group_vars/web_servers
git commit -m "Declare web server kernel tuning"
git push

# Repeat for database and application roles:
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh kernel-db
git add group_vars/database_servers
git commit -m "Declare database server kernel tuning"
git push

../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh kernel-app
git add group_vars/app_servers
git commit -m "Declare application server kernel tuning"
git push
```

Verify with `sysctl` on the target hosts. Each group has different optimal
values while the fleet baseline remains unchanged.

### Phase 7 — Group firewall rules

Uncomment the firewall overlays to open role-specific ports:

```bash
../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh firewall-web
git add group_vars/web_servers
git commit -m "Open HTTP/HTTPS for web servers"
git push

../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh firewall-db
git add group_vars/database_servers
git commit -m "Open PostgreSQL for database servers"
git push

../aap-demo-rhel-gitops-platform/utils/uncomment_demo.sh firewall-app
git add group_vars/app_servers
git commit -m "Open REST and metrics ports for app servers"
git push
```

### Phase 8 — Drift correction

Change one managed value manually on a target host, then launch the
reconciliation job without changing Git. AAP restores the value declared by the
state repository. This closes the GitOps loop for the audience.

## Uncomment Script Reference

The `uncomment_demo.sh` script in the platform repository accepts these phases:

| Phase | What it uncomments |
|---|---|
| `network` | `static_internal` sections in all host_vars network files |
| `storage-web` | `web_content_pool` in web_servers group and web host_vars |
| `storage-db` | `db_data_pool` in database_servers group and db host_vars |
| `storage-app` | `app_data_pool` in app_servers group and app host_vars |
| `kernel-web` | Kernel settings in web_servers group |
| `kernel-db` | Kernel settings in database_servers group |
| `kernel-app` | Kernel settings in app_servers group |
| `firewall-web` | HTTP/HTTPS rules in web_servers group |
| `firewall-db` | PostgreSQL rules in database_servers group |
| `firewall-app` | REST/metrics ports in app_servers group |
| `all` | Every phase above, in order |

Set `STATE_DIR` to override the default path to the state repository:

```bash
STATE_DIR=/path/to/state-repo ./utils/uncomment_demo.sh network
```

## Reset After a Demo

Restore all demo-modifiable files to their skeleton baseline from the
committed templates:

```bash
ansible-playbook playbooks/reset_demo.yml
git diff
git add -A && git commit -m "Reset demo state" && git push
```

Alternative: discard all local changes with `git reset --hard HEAD`.

## Variable Layers

| Layer | Location | Meaning |
|---|---|---|
| Fleet baseline | `group_vars/all/` | State shared by every RHEL host |
| Server role | `group_vars/<group>/` | State specific to a server group |
| Host identity | `host_vars/<host>/` | Stable host-specific state |

Every layer is merged via `combine(..., recursive=True)` in
`tasks/prepare_vars.yml`. The more specific layer wins.

## Layout

| Path | Purpose |
|---|---|
| `site.yml` | AAP reconciliation entry point |
| `tasks/prepare_vars.yml` | Converts Git-friendly dict declarations to role inputs |
| `group_vars/all/` | Fleet baseline (repos, time, SSH, sudo, firewall, storage, kernel) |
| `group_vars/web_servers/` | Web server overlays (commented-out for demo stages) |
| `group_vars/database_servers/` | Database server overlays |
| `group_vars/app_servers/` | Application server overlays |
| `host_vars/` | Host-specific MACs, static IPs, and disk bindings |
| `playbooks/reset_demo.yml` | Restore skeleton templates after a demo session |
| `templates/` | Skeleton copies of every demo-modifiable file |
| `collections/requirements.yml` | RHEL System Role dependency |
| `utils/` | Pre-demo population and uncomment scripts (in platform repo) |

## Platform Boundary

The companion `aap-demo-rhel-gitops-platform` repository owns KubeVirt VM
provisioning, SSH key creation, generated inventory, AAP credentials, and AAP
Configuration as Code. This repository remains safe to change during the live
human-driven GitOps portion of the event.

No generated IPs or MACs are committed here beyond the initial placeholders.
The platform's `utils/populate_host_vars.yml` playbook replaces them before the
demo starts.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `storage` role fails with missing packages | RHEL repos not yet registered | The `rhc` role runs first; check subscription-manager identity |
| `subscription-manager register` fails on second run | System already registered | The site.yml checks identity and skips re-registration |
| DNS resolution takes 10+ seconds | Transient resolver race | The site.yml retries `getent hosts subscription.rhsm.redhat.com` up to 6 times |
| Network role hangs or disconnects | Interface change interrupts SSH | The site.yml ends with `wait_for_connection` |
| MAC placeholders still present after populate | Discovery not re-run after new bootstrap | Re-run `utils/populate_host_vars.yml` from the platform repo |
