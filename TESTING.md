# Testing Guide for MTV Ansible Hook Examples

This guide explains how to test the Ansible hook examples for Migration Toolkit for Virtualization (MTV).

## Prerequisites

- Access to an OpenShift cluster with MTV installed
- `oc` CLI tool configured and logged in
- A source VM to migrate (VMware, OVA, or other supported source)
- SSH access to the source VM (for PreHook testing)

## Testing the PreHook - Preserve Interface Names

### Overview

This hook creates udev rules on the source VM **before migration** to ensure network interface names remain consistent after migration to OpenShift Virtualization.

### Why Test This?

Without this hook, interface names often change during migration (e.g., `eth0` → `ens3`), which breaks:
- DHCP configurations that depend on interface names
- Firewall rules tied to specific interfaces
- Application configurations referencing interface names
- Network bonding/teaming configurations

### Test Setup

#### 1. Create SSH Secret

The hook needs SSH access to the source VM. Create a secret with your SSH private key:

```bash
# Option A: Use the provided template (edit with your key first)
oc apply -f ansible/prehook-preserve-interface-names/ssh-secret.yaml

# Option B: Create from your existing SSH key
oc create secret generic vm-ssh-credentials \
  --from-file=key=~/.ssh/id_rsa \
  -n konveyor-forklift
```

**Important:** The SSH key must have passwordless access to the source VM as root.

#### 2. Verify VM Prerequisites

SSH to your source VM and verify:

```bash
# Check current interface names
ip addr show

# Example output:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500

# Note the interface names - we'll verify they're preserved after migration
```

#### 3. Create the Hook CR

```bash
# Apply the provided Hook CR
oc apply -f ansible/prehook-preserve-interface-names/hook-cr.yaml

# Verify it was created
oc get hooks -n konveyor-forklift
```

#### 4. Create Migration Plan with Hook

In the MTV UI:
1. Create a new migration plan
2. Select your source VM
3. In the "Hooks" section, add a hook:
   - **Hook:** `preserve-interface-names`
   - **Step:** `PreHook`
4. Complete the plan configuration

Or via YAML:

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata:
  name: test-interface-preservation
  namespace: konveyor-forklift
spec:
  provider:
    source:
      name: my-vmware-provider
      namespace: konveyor-forklift
    destination:
      name: host
      namespace: konveyor-forklift
  targetNamespace: default
  vms:
    - id: vm-123  # Replace with your VM ID
      hooks:
        - hook:
            name: preserve-interface-names
            namespace: konveyor-forklift
          step: PreHook
```

Apply it:
```bash
oc apply -f test-plan.yaml
```

### Running the Test

#### 1. Start the Migration

```bash
# Create a Migration CR to start the migration
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  name: test-migration-001
  namespace: konveyor-forklift
spec:
  plan:
    name: test-interface-preservation
    namespace: konveyor-forklift
EOF
```

#### 2. Monitor Hook Execution

```bash
# Watch for hook pod creation
watch -n 2 'oc get pods -n konveyor-forklift | grep hook'

# Once the hook pod appears, view its logs
oc logs -f $(oc get pods -n konveyor-forklift -o name | grep hook | head -1) -n konveyor-forklift
```

**Expected Output:**
```
TASK [Gather network interface information]
ok: [192.168.1.100]

TASK [Parse interface information]
ok: [192.168.1.100] => (item=eth0:52:54:00:12:34:56)
ok: [192.168.1.100] => (item=eth1:52:54:00:ab:cd:ef)

TASK [Create udev rules file to preserve interface names]
changed: [192.168.1.100]

TASK [Summary of changes]
ok: [192.168.1.100] => {
    "msg": [
        "✓ Created udev rules for 2 network interfaces",
        "✓ Rules location: /etc/udev/rules.d/70-persistent-net.rules",
        ...
    ]
}
```

#### 3. Verify Hook Created udev Rules

SSH to the source VM (while migration is paused or before it completes):

```bash
ssh root@<source-vm-ip>

# Check the udev rules were created
cat /etc/udev/rules.d/70-persistent-net.rules

# Expected content:
# SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="52:54:00:12:34:56", NAME="eth0"
# SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="52:54:00:ab:cd:ef", NAME="eth1"

# Run the verification script
/root/verify-interface-names.sh
```

#### 4. Verify After Migration Completes

Once the migration completes and the VM is running on OpenShift:

```bash
# Get the VM's IP in OpenShift
oc get vmi -n <target-namespace>

# SSH to the migrated VM
ssh root@<new-vm-ip>

# Verify interface names are preserved
ip addr show

# Expected: Same interface names as before (eth0, eth1, etc.)
# NOT changed to ens3, ens4, etc.

# Run verification script
/root/verify-interface-names.sh
```

### Test Success Criteria

✅ **PreHook pod completes successfully** (status: Completed)  
✅ **udev rules file created** on source VM  
✅ **Interface names preserved** after migration  
✅ **Network connectivity works** after migration  
✅ **No errors in hook pod logs**

### Test Failure Scenarios to Check

❌ **SSH connection fails** → Check SSH secret has correct key  
❌ **Hook pod fails** → Check logs for Ansible errors  
❌ **Interface names still change** → Verify udev rules syntax  
❌ **Network broken after migration** → Check if udev rules match correct MACs

---

## Testing the PostHook Example

### Overview

The example posthook (**posthook-restore-network**) runs after migration and only loads MTV metadata and logs "Post hook ran successfully." It does not touch networking or SSH to the VM.

### Test Setup

#### 1. Create the Hook CR

```bash
oc apply -f ansible/posthook-restore-network/hook-cr.yaml
oc get hooks -n konveyor-forklift
```

#### 2. Add to Migration Plan

Add the hook with `step: PostHook`:

```yaml
vms:
  - id: vm-123
    hooks:
      - hook:
          name: restore-network
          namespace: konveyor-forklift
        step: PostHook
```

### Running the Test

Run a migration that uses this plan. The posthook runs after the VM is migrated.

### Test Success Criteria

✅ **PostHook job completes successfully**  
✅ **Job logs show**: "Post hook ran successfully."

---

## Customizing for Your Environment

### Modify SSH Secret Name

If your SSH secret has a different name, edit the playbook:

```yaml
- name: Retrieve SSH credentials from Kubernetes Secret
  k8s_info:
    api_version: v1
    kind: Secret
    name: your-secret-name  # Change this
    namespace: konveyor-forklift
```

### Use Different SSH User

If your VM doesn't allow root SSH, change:

```yaml
- name: Add target VM to Ansible inventory
  add_host:
    name: "{{ workload.vm.ipaddress }}"
    ansible_user: your-user  # Change from 'root'
    ansible_become: yes      # Add this to use sudo
```

### Test with Multiple VMs

Create a plan with multiple VMs to verify the hook runs for each VM independently.

---

## Troubleshooting

### Hook Pod Fails with "Connection refused"

**Cause:** SSH credentials are incorrect or VM doesn't allow SSH  
**Fix:** Verify SSH key in secret matches VM's authorized_keys

### Hook Pod Shows "Permission denied"

**Cause:** SSH user doesn't have permissions  
**Fix:** Use root or add `ansible_become: yes` to use sudo

### Interface Names Still Change

**Cause:** udev rules not applied before migration  
**Fix:** Verify hook is set as PreHook (not PostHook), check udev rules file exists on source VM

### Can't Access VM After Migration

**Cause:** udev rules might have wrong MAC addresses  
**Fix:** Check MAC addresses in udev rules match VM's actual MACs

---

## Questions or Issues?

For questions about these examples or issues during testing:
1. Check the hook pod logs: `oc logs <hook-pod> -n konveyor-forklift`
2. Verify prerequisites are met (SSH access, correct namespace, etc.)
3. Review the playbook for environment-specific adjustments needed

---

## Additional Resources

- [MTV Documentation](https://access.redhat.com/documentation/en-us/migration_toolkit_for_virtualization/)
- [Ansible Documentation](https://docs.ansible.com/)
- [udev Rules Documentation](https://www.freedesktop.org/software/systemd/man/udev.html)

