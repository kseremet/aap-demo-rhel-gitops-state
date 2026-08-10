![Status](https://img.shields.io/badge/Status-Scaffold-yellow)
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

## Workflow

```text
Human edits state -> Commit and push -> AAP project sync
    -> RHEL System Roles reconcile -> Fleet converges
```

## Quick Start

This split is currently a scaffold. The commands below are the intended
implementation and event script; the current `site.yml` only validates that the
state scaffold loads until the remaining System Role tasks are added.

Complete the platform repository bootstrap first. It must create the RHEL VMs,
populate the visible AAP inventory, create the Machine Credential, and configure
the AAP Project that points to this repository.

### 1. Prepare a local checkout

```bash
cd aap-demo-rhel-gitops-state
cp ansible.cfg.example ansible.cfg
ansible-galaxy collection install \
  -r collections/requirements.yml \
  -p collections
```

The state repository does not contain the management IP inventory. For local
runs, pass the generated platform inventory explicitly:

```bash
ansible-inventory \
  -i ../aap-demo-rhel-gitops-platform/environments/lab/inventory.yml \
  --graph
```

### 2. Run the initial baseline

The first reconciliation applies only the fleet-wide baseline. This is the
starting point for the live demo: every host receives the same subscription,
repository, time synchronization, access, firewall, storage, and kernel baseline.

```bash
ansible-playbook \
  -i ../aap-demo-rhel-gitops-platform/environments/lab/inventory.yml \
  site.yml
```

In the event presentation, launch this same operation from the AAP
reconciliation job template after committing the baseline state.

### 3. Demonstrate group-specific networking

Edit the network declarations in the state repository. Uncomment the static
internal interface in the fleet or group networking file, then define the
host-specific MAC and address values under `host_vars/<host>/network.yml`.

```bash
git diff
git add group_vars host_vars
git commit -m "Declare internal network for web servers"
git push
```

Launch the AAP reconciliation job again. Explain that the commit is the change
event and the System Role is the reconciler; no administrator needs to run a
manual `nmcli` command on the target host.

### 4. Demonstrate group-specific storage

Enable the web, database, or application storage overlay in the matching
`group_vars/<group>/storage.yml`. Bind the required disk by stable path in the
matching `host_vars/<host>/storage.yml`.

```bash
git add group_vars host_vars
git commit -m "Declare role-specific storage"
git push
```

Run reconciliation and verify the declared volume and mount point on the
appropriate hosts. Other groups should retain only the fleet baseline.

### 5. Demonstrate kernel tuning

Add or uncomment a kernel setting in the relevant group overlay, for example a
database shared-memory or web connection backlog setting. Use a host overlay
only when one host has a special requirement.

```bash
git add group_vars host_vars
git commit -m "Declare group kernel tuning"
git push
```

Run the job again and verify the effective value with `sysctl` on the target.
The Git file remains the source of truth; the command is only verification.

### 6. Demonstrate drift correction

Change one managed value manually on a target host, then run the reconciliation
job without changing Git. AAP should restore the value declared by the state
repository. This closes the GitOps loop for the audience.

### 7. Develop locally, demonstrate through AAP

Use local runs to validate syntax and behavior quickly:

```bash
ansible-playbook \
  -i ../aap-demo-rhel-gitops-platform/environments/lab/inventory.yml \
  site.yml --syntax-check

ansible-playbook \
  -i ../aap-demo-rhel-gitops-platform/environments/lab/inventory.yml \
  site.yml
```

Use AAP for the audience-facing workflow: project sync, job launch, output, and
repeatable reconciliation after each Git commit.

## Variable Layers

| Layer | Location | Meaning |
|---|---|---|
| Fleet baseline | `group_vars/all/` | State shared by every RHEL host |
| Server role | `group_vars/web_servers/` | State for a server group |
| Host identity | `host_vars/<host>/` | Stable host-specific state |

## Layout

| Path | Purpose |
|---|---|
| `site.yml` | AAP reconciliation entry point |
| `tasks/prepare_vars.yml` | Converts Git-friendly declarations to role inputs |
| `group_vars/` | Fleet and server-group desired state |
| `host_vars/` | Stable host-specific desired state |
| `collections/requirements.yml` | RHEL System Role dependency |
| `docs/` | Demo procedures and verification |
| `tests/` | State validation tests |

## Platform Boundary

The companion `aap-demo-rhel-gitops-platform` repository owns KubeVirt VM
provisioning, SSH key creation, generated inventory, AAP credentials, and AAP
Configuration as Code. This repository should remain safe to change during the
live human-driven GitOps portion of the event.

## Status

This repository is currently a design scaffold. The quick-start sequence
documents the target event script; the remaining System Role declarations and
role invocations are being added incrementally. The existing
`aap-demo-rhel-gitops` demo remains intact while desired-state content is moved
here incrementally.
