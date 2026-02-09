# Quick Start: Test a Migration with Ansible Hooks

You do **not** need to build a new Forklift controller. Hooks are defined by Hook CRs and referenced in your Plan; the default controller runs them.

## 1. Prerequisites

- MTV (Forklift) installed and a working migration (source + destination providers, plan, migration).
- **SSH access** to the source VM as **root** (passwordless key-based).
- The cluster can reach the source VM’s IP (for PreHook) and later the migrated VM (for PostHook).

## 2. Create the SSH secret (once per namespace)

The hook runs in the **MTV operator namespace** (e.g. `openshift-mtv` or `konveyor-forklift`). The secret must be in **that** namespace.

```bash
# Replace with your MTV namespace if different (e.g. openshift-mtv)
export MTV_NS=openshift-mtv

# Create secret from your SSH private key (must work as root on the VM)
kubectl create secret generic vm-ssh-credentials \
  --from-file=key=/path/to/your/private_key \
  -n $MTV_NS
```

Use the same key you would use to `ssh root@<source-vm-ip>`.

## 3. Apply the Hook CR(s)

From the repo root:

```bash
# PreHook: preserve interface names on source VM
oc apply -f ansible/prehook-preserve-interface-names/hook-cr.yaml

# PostHook example: logs success only (no networking)
oc apply -f ansible/posthook-restore-network/hook-cr.yaml
```

If your Hook CRs use a different namespace, edit `metadata.namespace` in each `hook-cr.yaml` (or use `oc apply -f ... -n $MTV_NS` after setting `MTV_NS`).

Verify:

```bash
oc get hooks -n $MTV_NS
```

## 4. Add hooks to your Plan

You don’t copy the playbook into the UI. You **reference** the Hook by name in the Plan. MTV then runs the playbook that is stored in the Hook CR.

### Option A: MTV UI

1. Open your migration plan (or create one).
2. Select the VM(s) to migrate.
3. Find the **Hooks** (or **Ansible hooks**) section for that VM.
4. Add a hook:
   - **Hook:** `preserve-interface-names` (or `restore-network` for posthook).
   - **Namespace:** same as where you applied the Hook CR (e.g. `openshift-mtv`).
   - **Step:** `PreHook` or `PostHook` as appropriate.
5. Save the plan.

### Option B: YAML (Plan spec)

Under `spec.vms`, add `hooks` to the VM(s) that should run the playbook:

```yaml
spec:
  vms:
    - id: "<your-vm-id>"    # Same ID you use for a normal migration
      hooks:
        - hook:
            name: preserve-interface-names
            namespace: openshift-mtv
          step: PreHook
        # Optional: run after migration
        - hook:
            name: restore-network
            namespace: openshift-mtv
          step: PostHook
```

Replace `id` with your VM ID from the provider. Replace `namespace` if you use e.g. `openshift-mtv`.

Then apply the plan:

```bash
oc apply -f your-plan.yaml
```

## 5. Run the migration

Create a **Migration** that references this plan (same as a migration without hooks):

```bash
oc apply -f - <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  name: my-migration-with-hooks
  namespace: openshift-mtv
spec:
  plan:
    name: <your-plan-name>
    namespace: openshift-mtv
EOF
```

Or start the migration from the MTV UI using the plan you edited.

## 6. Watch the hook run

- **PreHook:** runs before the VM is migrated (connects to source VM by IP from the plan).
- **PostHook:** runs after the VM is migrated (connects to the migrated VM).

```bash
# List hook jobs/pods
oc get jobs,pods -n $MTV_NS | grep -i hook

# Stream logs (replace <hook-pod-or-job-name> with the actual name)
oc logs -f job/<hook-job-name> -n $MTV_NS
# or
oc logs -f <hook-pod-name> -n $MTV_NS
```

Successful PreHook output will show Ansible tasks running and then SSH to the VM (e.g. “Gather network interface information”, “Create udev rules …”, “Write network state file …”).

## Summary

| Step | What you do |
|------|------------------|
| 1 | Create `vm-ssh-credentials` secret in MTV namespace. |
| 2 | `oc apply -f ansible/.../hook-cr.yaml` for each hook you want. |
| 3 | In the Plan (UI or YAML), add `hooks` for the VM with hook name + namespace + step. |
| 4 | Start a Migration that uses that Plan (UI or `oc apply` Migration). |
| 5 | Check hook jobs/pods and logs in the MTV namespace. |

You do **not** paste the playbook into the UI; the Playbook is in the Hook CR. You only attach the hook by **name** and **namespace** in the Plan.

For more detail and troubleshooting, see [TESTING.md](../TESTING.md).
