# MTV Ansible Hook Examples

This directory contains simple, working examples of Ansible playbooks for Migration Toolkit for Virtualization (MTV) hooks.

## What are MTV Hooks?

MTV hooks let you run Ansible playbooks at specific points during VM migration:
- **PreHook**: Runs before migration starts (e.g., prepare the VM)
- **PostHook**: Runs after migration completes (e.g., restore network, install monitoring)

**Important:** Hooks run inside a bare Ansible (hook-runner) container that does *not* have tools like `firewalld`, `nmcli`, `dbus`, or `getent`. Playbooks must be **container-safe** on localhost (use `lookup('env', 'HOME')` instead of `getent`, avoid firewall/NM modules on localhost) and perform all VM-specific work **via SSH** so that commands run on the VM where those tools may exist.

## Quick Start

### 1. Set up SSH access to your VMs

Create a Kubernetes secret with your SSH private key:

```bash
kubectl create secret generic vm-ssh-credentials \
  --from-file=key=/path/to/your/private/key \
  -n openshift-mtv
```

### 2. Choose an example

- **prehook-preserve-interface-names/**: Capture interface/MAC/NM/DHCP state and create udev rules on the source VM (SSH-based)
- **posthook-restore-network/**: Restore udev and NetworkManager state on the target VM from pre-hook state file (SSH-based)
- **posthook-monitoring/**: Install node_exporter monitoring on the target VM (SSH-based; firewall steps optional)

### 3. Apply the Hook CR

```bash
# PreHook: preserve interface names + NM/DHCP state
kubectl apply -f prehook-preserve-interface-names/hook-cr.yaml

# PostHook (optional): restore network state after migration
kubectl apply -f posthook-restore-network/hook-cr.yaml
```

### 4. Reference the hook in your migration Plan

You don't paste the playbook into the UI—you reference the Hook by name. In your Plan (UI or YAML), add hooks to the VM:

```yaml
spec:
  vms:
    - id: vm-123
      hooks:
        - hook:
            namespace: openshift-mtv
            name: preserve-interface-names
          step: PreHook
        - hook:
            namespace: openshift-mtv
            name: restore-network
          step: PostHook
```

See [QUICKSTART-TEST-HOOKS.md](QUICKSTART-TEST-HOOKS.md) for a full test walkthrough.

## How It Works

1. MTV creates a Job in the cluster when the hook is triggered
2. The job runs the hook-runner (Ansible) container with your playbook
3. The playbook runs a small localhost play (load plan/workload, fetch SSH key from K8s), then adds the VM to inventory and runs the rest on the VM via SSH
4. All VM-specific commands (udev, nmcli, firewall, etc.) execute on the VM over SSH, not in the container
5. Migration continues after hook completes

For network preservation: use the **prehook-preserve-interface-names** so the source VM gets udev rules and a state file (`/root/.mtv-network-state.json`) with interface, MAC, NM UUID, and DHCP IP. After migration, use the **posthook-restore-network** so the target VM re-applies udev and NM from that state file (which migrated with the VM).

## Customizing for Your Environment

### Update VM connection details

Each playbook connects to the VM using information from MTV. The VM's IP address is available in the `workload.vm.ipaddress` variable.

### Modify SSH credentials

Update the secret name in playbooks:
```yaml
- k8s_info:
    api_version: v1
    kind: Secret
    name: vm-ssh-credentials  # Change this to your secret name
    namespace: openshift-mtv
  register: ssh_creds
```

### Change the SSH user

Update the `ansible_user` in playbooks:
```yaml
- add_host:
    name: "{{ workload.vm.ipaddress }}"
    ansible_user: root  # Change to your SSH user
    groups: target_vms
```

## Creating Your Own Playbook

1. Write your Ansible playbook (see examples)
2. Encode it to base64:
   ```bash
   ./scripts/encode-playbook.sh your-playbook.yml
   ```
3. Create a Hook CR with the encoded playbook
4. Apply it and reference it in your Plan

## Troubleshooting

### Hook job fails to start
- Check that the hook-runner image is accessible: `quay.io/konveyor/hook-runner:latest`
- Verify ServiceAccount permissions (default: `forklift-controller`)

### Cannot connect to VM
- Verify SSH credentials are correct
- Check that the VM's IP is accessible from the cluster
- Ensure the SSH key has proper permissions (0600)
- Verify the SSH user exists on the VM

### Playbook tasks fail
- Check hook job logs: `kubectl logs -n openshift-mtv job/<hook-job-name>`
- Verify the VM has required packages/dependencies
- Check that the user has sudo permissions if needed

## Requirements

- MTV 2.6 or later
- SSH access to source VMs
- Network connectivity from the cluster to VMs

## Contributing

Found an issue or have a useful example to share? Please open an issue or PR in the main forklift repository.

