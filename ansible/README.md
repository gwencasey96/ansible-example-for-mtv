# MTV Ansible Hook Examples

This directory contains simple, working examples of Ansible playbooks for Migration Toolkit for Virtualization (MTV) hooks.

## Goals

1. **DHCP preserved after migration**  
   After migration, the migrated VM should have interface names identical to the source VM, interfaces up, and an IP from DHCP on the **destination** (pod) network. Source and destination networks are separate—"preserved" means the VM gets DHCP on the new network, not the same IP as on the source.

2. **Provide an example PreHook and an example PostHook**  
   You need at least one example of each hook type. They do not have to be related; each example just needs to run correctly at its step.

## Expected workflow (DHCP / interface preservation)

- **Problem:** After migration, interface is down and no IP.
- **Add a PreHook** that injects udev rules on the source VM (so interface names are bound to MACs).
- **Migrate the VM.**
- **Log in to the migrated VM** and run `ip a`.
- **Success criteria:**
  - Interface names are identical to the source VM.
  - Interfaces are up.
  - The VM gets an IP address from DHCP on the pod network.

The **prehook-preserve-interface-names** example does the udev injection on the source; the migrated disk carries those rules, so the migrated VM keeps the same interface names and can get DHCP on the destination network.

## What are MTV Hooks?

MTV hooks let you run Ansible playbooks at specific points during VM migration:
- **PreHook**: Runs before migration starts (e.g., prepare the source VM)
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

- **PreHook example:** **prehook-preserve-interface-names/** — Injects udev rules on the source VM so interface names (e.g. eth0) are bound to MAC addresses. After migration, the VM keeps the same interface names and can get DHCP on the pod network.
- **PostHook example:** **posthook-restore-network/** — Loads MTV metadata and logs "Post hook ran successfully." It does not touch networking; use it as a minimal posthook example.

### 3. Apply the Hook CR

```bash
# PreHook: preserve interface names
kubectl apply -f prehook-preserve-interface-names/hook-cr.yaml

# PostHook example: logs success only
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
3. Prehook: playbook loads plan/workload, fetches SSH key, then runs tasks on the source VM via SSH (e.g. udev rules)
4. Posthook example: playbook loads plan/workload and logs "Post hook ran successfully" (no SSH, no networking)
5. Migration continues after hook completes

For **DHCP preservation**: use the **prehook-preserve-interface-names** so the source VM gets udev rules. The migrated disk carries those rules; on the destination, interface names stay the same and the VM can get an IP from DHCP on the pod network. The **posthook-restore-network** is a minimal posthook example that only logs success.

## Customizing for Your Environment

### Update VM connection details (PreHook only)

The prehook connects to the source VM using information from MTV. The VM's IP address is available in the `workload.vm.ipaddress` variable. The example posthook does not connect to any VM.

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

### "A playbook must be a list of plays, got ... AnsibleMapping ... spec:"
This means the **Hook CR’s `spec.playbook`** contains the wrong content. Ansible received YAML that starts with `spec:` (a Kubernetes resource) instead of a valid playbook (a list of plays starting with `- name:` or `---` then `- name:`).

**Cause:** The base64 value in `spec.playbook` was set from the wrong source—for example the full Hook CR YAML (or another K8s resource) instead of only the playbook file.

**Fix:**
1. Identify the Hook CR your plan uses (from the failed job name or Plan `spec.vms[].hooks[]`). That Hook’s `spec.playbook` must contain **only** the base64-encoded **playbook** (the contents of `playbook.yml`), not the full Hook CR or any other Kubernetes YAML.
2. Regenerate the correct base64 and update the Hook:
   ```bash
   cd ansible
   ./scripts/encode-playbook.sh prehook-preserve-interface-names/playbook.yml
   ```
   Copy the printed base64 string (between the `---` lines) into that Hook CR’s `spec.playbook`. If you use the repo’s Hook, edit `prehook-preserve-interface-names/hook-cr.yaml` and re-apply: `kubectl apply -f prehook-preserve-interface-names/hook-cr.yaml`.
3. If you created the Hook via the UI with “paste playbook”, paste only the **raw playbook YAML** (or its base64), not the Hook CR YAML.
4. Verify: decode the stored value and confirm the first line is `---` and the next meaningful line is `- name: ...`:
   ```bash
   echo '<your-base64-string>' | base64 -d | head -5
   ```

### "A playbook must be a list of plays, got ... AnsibleUnicode ..." and the offending line shows base64 (LS0t...)
The file at `/tmp/hook/playbook.yml` contains the **raw base64 string** instead of decoded YAML. This usually means the UI **base64-encodes** whatever you paste: if you pasted base64, it was encoded again, so after one decode the file still has base64.

**Fix:** In the MTV UI Ansible playbook field, paste the **raw playbook YAML** (the decoded content), not the base64. Copy the entire contents of `ansible/prehook-preserve-interface-names/playbook.yml` (from `---` through the last line) and paste that. The UI will encode it once; the controller will decode and get valid YAML.

## Requirements

- MTV 2.6 or later
- SSH access to source VMs
- Network connectivity from the cluster to VMs

## Contributing

Found an issue or have a useful example to share? Please open an issue or PR in the main forklift repository.

