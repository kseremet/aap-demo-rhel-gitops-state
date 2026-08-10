# RHEL GitOps State Demo

This folder is the human-maintained desired-state side of the RHEL GitOps event
demo **From Admin to Maintainer: Agentic Ops and GitOps for RHEL**.

## Purpose

Demonstrate a purely human GitOps workflow. An administrator changes declared
RHEL state in Git, commits and pushes it, and AAP reconciles the fleet with RHEL
System Roles.

## Ownership rules

- RHEL desired state belongs here.
- `group_vars/all/` contains fleet-wide intent.
- `group_vars/<group>/` contains role-specific intent.
- `host_vars/<host>/` contains stable host-specific intent.
- Host names are stable identities; management IP addresses are supplied by the
  platform-owned AAP inventory.
- Do not add KubeVirt provisioning, VM deletion, SSH key generation, OpenShift
  tokens, or generated discovery output here.

## Demonstration narrative

1. Show the fleet and its groups in AAP.
2. Change one desired-state file in Git.
3. Commit and push the change.
4. Let AAP sync the project and run reconciliation.
5. Verify the RHEL host converges to the declared state.
6. Repeat with baseline, group, and host-specific changes.

## Companion repository

The platform/bootstrap repository is `aap-demo-rhel-gitops-platform`. It creates
the VMs, discovers their visible management IPs, creates the AAP credentials and
inventory, and configures the AAP project that points to this repository.

## Current status

This is a new scaffold. The existing `../aap-demo-rhel-gitops` demo is preserved
as-is and is not modified by this repository.
