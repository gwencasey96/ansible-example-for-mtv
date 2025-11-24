# MTV Ansible Hook Examples

This repository contains practical Ansible playbook examples for use with Migration Toolkit for Virtualization (MTV) hooks.

## What are MTV Hooks?

MTV hooks allow you to run custom Ansible playbooks at specific points during VM migration:
- **PreHooks**: Run before migration starts (on the source VM)
- **PostHooks**: Run after migration completes (on the migrated VM)

## Examples Included

### 1. PreHook: Preserve Network Interface Names
**Location**: `ansible/prehook-preserve-interface-names/`

This playbook creates udev rules to preserve network interface names during migration. This is critical for VMs using DHCP where interface name changes would break network configuration.

**What it does**:
- Discovers all network interfaces and their MAC addresses on the source VM
- Creates persistent udev rules to bind interface names to MAC addresses
- Ensures interface names remain consistent after migration (e.g., eth0 stays eth0)
- Includes verification script for post-migration testing

**Use case**: Prevents network configuration breakage for DHCP-configured VMs

### 2. PostHook: Install Prometheus Node Exporter
**Location**: `ansible/posthook-node-exporter/`

This playbook installs and configures Prometheus node_exporter for monitoring after migration completes.

**What it does**:
- Installs node_exporter monitoring agent
- Configures systemd service
- Sets up firewall rules
- Demonstrates post-migration automation pattern

## Getting Started

See [TESTING.md](TESTING.md) for comprehensive instructions on:
- Prerequisites and setup
- How to use these examples in your migrations
- Step-by-step testing procedures
- Expected outputs and success criteria
- Troubleshooting common issues
- Customization examples

## Quick Start

1. Create SSH credentials secret (see `ansible/*/ssh-secret.yaml`)
2. Encode your playbook: `base64 -i playbook.yml`
3. Create Hook CR with encoded playbook (see `ansible/*/hook-cr.yaml`)
4. Reference the hook in your MTV migration plan

## Documentation

- [MTV Hooks Documentation](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/)
- [Detailed Testing Guide](TESTING.md)

## Contributing

These examples are meant to serve as reference implementations. Feel free to customize them for your specific migration needs.

## Related

- Main MTV Project: [kubev2v/forklift](https://github.com/kubev2v/forklift)
- Issue: [MTV-3660](https://issues.redhat.com/browse/MTV-3660)

