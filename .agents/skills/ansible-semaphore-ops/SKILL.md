---
name: ansible-semaphore-ops
description: Build and troubleshoot safe Ansible playbooks run from Semaphore against Kubernetes or Linux nodes. Use when writing Semaphore playbooks, handling static inventory/SSH/Python target issues, using raw for old managed nodes, or creating guarded node maintenance tasks such as CRI image pruning or Linux cache cleanup.
license: MIT
compatibility: OpenCode project-local skill for Ansible, Semaphore, Kubernetes node operations, and Linux maintenance playbooks.
metadata:
  audience: ops
  workflow: semaphore-ansible
---

# Ansible Semaphore Ops

## Purpose

Use this skill to create or debug Ansible playbooks that run from Semaphore against Linux or Kubernetes nodes, especially when the controller is modern but managed nodes are old, minimal, or operationally sensitive.

This is not a generic Ansible style guide. It captures the hard-won patterns from Semaphore-driven node operations: static inventory surprises, non-default SSH ports, target Python incompatibility, `raw` module tradeoffs, CRI/containerd cleanup, cache cleanup, dry-run defaults, canary execution, and shell portability.

## When To Use

Use this skill when the task mentions any of these:

- Semaphore running an Ansible template or project repository playbook.
- A playbook path under `/tmp/semaphore/repository/...`.
- `Could not match supplied host pattern` or static inventory confusion.
- SSH failures from Semaphore to target nodes, especially non-default ports.
- `Ansible requires Python 3.9 or newer on the target`.
- Managed nodes with Python 3.8 or no usable Python.
- `ansible.builtin.raw` for bootstrap, diagnostics, or maintenance.
- Kubernetes node maintenance through Ansible.
- CRI/containerd image cleanup with `crictl`.
- Linux memory cache cleanup with `/proc/sys/vm/drop_caches`.

Do not use this skill for broad Ansible role design, app deployment, cloud provisioning, or Windows targets unless the same Semaphore/node-maintenance constraints are present.

## Core Principles

1. Verify what Semaphore actually runs before editing the wrong file.
2. Treat inventory, SSH, authentication, remote Python, and task command failures as separate layers.
3. For old managed nodes, use `raw` intentionally and document why.
4. For operational cleanup, default to preview mode and require an explicit execute variable.
5. Run one node at a time for disruptive maintenance with `serial: 1`.
6. Preflight privileges and required tools before doing work.
7. Print enough raw diagnostic output to explain failures from Semaphore logs.
8. Use canary execution before full fleet execution.
9. Never delete runtime directories manually.
10. Never hide failed cleanup behind a successful-looking debug task.

## Semaphore Debug Flow

When a Semaphore task fails, walk the layers in this order.

### 1. Confirm the Executed Playbook

Semaphore may run a different file than the one edited locally. Check the task log for the playbook name and inspect the pod repository path.

Common symptom:

```text
[WARNING]: Could not match supplied host pattern, ignoring: test
```

Interpretation:

- The playbook says `hosts: test`.
- Semaphore selected a static inventory that does not contain a `test` group.
- The task can still have valid hosts, but the play's host pattern does not match them.

Safe default for Semaphore templates that supply inventory externally:

```yaml
- name: Node maintenance task
  hosts: all
  gather_facts: false
```

Use a specific group only after verifying that Semaphore's selected inventory defines that group.

### 2. Separate SSH Port From Authentication

If logs show port 22 refused, test the configured node port from the Semaphore pod or controller environment.

Expected interpretation:

- `Connection refused` means wrong port, service down, or firewall.
- `Permission denied (publickey,password)` means the port is reachable and the remaining problem is credentials/user.

For non-default SSH ports, prefer inventory or play vars:

```yaml
vars:
  ansible_port: 36633
```

### 3. Separate Target Python From Command Failure

`gather_facts: false` only skips fact gathering. It does not make normal modules Python-free.

Most POSIX Ansible modules, including `command` and `shell`, require a Python interpreter on the managed node. With modern `ansible-core`, old targets may fail before the command runs.

Common symptom:

```text
Ansible requires Python 3.9 or newer on the target. Current version: 3.8.10
```

Resolution choices:

- Best long-term: install or select Python 3.9+ on managed nodes.
- If Python 3.9+ already exists: set `ansible_python_interpreter` to that path.
- For diagnostics or bootstrap: use `ansible.builtin.raw` and `gather_facts: false`.

Do not switch from `command` to `shell` expecting to avoid Python. Use `raw` only when the Python dependency is the reason.

## Raw Module Pattern

Use `raw` for minimal remote commands on old or Python-incompatible nodes.

```yaml
- name: Run diagnostic command without target Python
  ansible.builtin.raw: ip addr || ifconfig
  register: network_result
  changed_when: false
  failed_when: network_result.rc != 0

- name: Show diagnostic output
  ansible.builtin.debug:
    var: network_result.stdout_lines
```

Guidelines:

- Set `gather_facts: false` when using `raw` to work around missing/old Python.
- Always `register` output for operational visibility.
- Set `changed_when: false` for read-only diagnostics.
- Use `failed_when` for commands whose return code matters.
- Keep raw command blocks POSIX-sh compatible unless the target shell is known.

## Zsh-Safe Privilege Pattern

Do not store a command with arguments in a scalar variable like `SUDO="sudo -n"` and then run `$SUDO crictl ...`. In zsh this can be interpreted as a single command named `sudo -n`.

Use a function instead:

```sh
run_as_root() {
  if [ "$(id -u)" -eq 0 ]; then
    "$@"
  else
    sudo -n "$@"
  fi
}
```

Preflight privilege before cleanup:

```sh
if [ "$(id -u)" -eq 0 ]; then
  :
elif command -v sudo >/dev/null 2>&1 && sudo -n true >/dev/null 2>&1; then
  :
else
  echo "FAIL: current user must be root or have passwordless sudo"
  exit 20
fi
```

## Safe Maintenance Playbook Skeleton

Use this shape for node cleanup or other sensitive maintenance:

```yaml
---
- name: Safe node maintenance
  hosts: all
  gather_facts: false
  serial: 1

  vars:
    ansible_port: 36633
    maintenance_execute: false

  tasks:
    - name: Preflight | verify prerequisites
      ansible.builtin.raw: |
        set -eu
        # privilege/tool/runtime checks here
      register: preflight_result
      changed_when: false
      failed_when: preflight_result.rc != 0

    - name: Preview | show current state
      ansible.builtin.raw: |
        set -eu
        # read-only inspection here
      register: preview_result
      changed_when: false
      failed_when: preview_result.rc != 0

    - name: Safety | explain dry-run mode
      ansible.builtin.raw: |
        echo "No changes performed. Re-run with -e maintenance_execute=true."
      register: safety_result
      changed_when: false
      failed_when: false
      when: not (maintenance_execute | bool)

    - name: Cleanup | perform explicit operation
      ansible.builtin.raw: |
        set -eu
        # destructive or cache-dropping action here
      register: cleanup_result
      changed_when: cleanup_result.rc == 0
      failed_when: cleanup_result.rc != 0
      when: maintenance_execute | bool

    - name: Verify | show state after operation
      ansible.builtin.raw: |
        set -eu
        # read-only verification here
      register: verify_result
      changed_when: false
      failed_when: verify_result.rc != 0
```

## CRI/containerd Image Cleanup

For Kubernetes nodes using CRI/containerd, prefer `crictl rmi --prune` to remove images not used by containers.

Required safeguards:

- Default `cleanup_execute: false`.
- Use `serial: 1`.
- Preflight `crictl` exists.
- Preflight `crictl info` can access the runtime.
- Print raw `crictl info` output when runtime access fails.
- Use a configurable timeout for slow image deletion.
- Preview `df -h` and `crictl images` before cleanup.
- Verify disk and image state after cleanup.

Short pattern:

```yaml
vars:
  cleanup_execute: false
  crictl_timeout: 60s
```

```sh
CRI_INFO_OUTPUT=$(run_as_root crictl --timeout {{ crictl_timeout }} info 2>&1) || {
  echo "FAIL: crictl cannot access the CRI runtime"
  echo
  echo "crictl info output:"
  echo "$CRI_INFO_OUTPUT"
  echo
  echo "Hint: check /etc/crictl.yaml and the containerd socket path on this node."
  exit 22
}

run_as_root crictl --timeout {{ crictl_timeout }} images
run_as_root crictl --timeout {{ crictl_timeout }} rmi --prune
```

Common `crictl` notes:

- Missing `/etc/crictl.yaml` can be a warning if default endpoints still work.
- Common containerd endpoints include `unix:///run/containerd/containerd.sock` and `unix:///var/run/containerd/containerd.sock`.
- `DeadlineExceeded` during delete can be timeout/runtime pressure. Verify whether the reported image ID still exists before declaring failure.

Never use these as a substitute for CRI pruning:

- `docker system prune` on containerd nodes.
- `ctr` without understanding namespaces and Kubernetes usage.
- `rm -rf /var/lib/containerd` or runtime storage directories.

## Linux Memory Cache Cleanup

Dropping caches is a cache reclaim operation, not a process-memory cleanup. It can discard clean page cache, dentries, and inodes, and may temporarily increase I/O/CPU after the cache is rebuilt.

Required safeguards:

- Default `memory_cleanup_execute: false`.
- Use `serial: 1`.
- Do not kill processes.
- Do not clear swap unless explicitly requested and separately justified.
- Do not restart services.
- Do not persist sysctl changes.
- Preflight write access to `/proc/sys/vm/drop_caches`.
- Restrict `drop_caches_value` to `1`, `2`, or `3`.
- Run `sync` before writing `drop_caches`.
- Preview and verify with `free -h` and selected `/proc/meminfo` fields.

Short pattern:

```yaml
vars:
  memory_cleanup_execute: false
  drop_caches_value: 3
```

```sh
case "{{ drop_caches_value }}" in
  1|2|3) ;;
  *)
    echo "FAIL: drop_caches_value must be 1, 2, or 3"
    exit 22
    ;;
esac

run_as_root sync
if [ "$(id -u)" -eq 0 ]; then
  echo "{{ drop_caches_value }}" > /proc/sys/vm/drop_caches
else
  printf '%s\n' "{{ drop_caches_value }}" | sudo -n tee /proc/sys/vm/drop_caches >/dev/null
fi
```

## Validation Workflow

Validate in this order:

1. Read the actual playbook content that Semaphore will run.
2. Run YAML diagnostics if available.
3. Run Ansible syntax check.
4. Run preview mode against one canary node.
5. Run execute mode against one canary node.
6. Verify node/workload health externally if the operation affects Kubernetes nodes.
7. Run full fleet only after canary passes.

Command pattern:

```bash
ansible-playbook --syntax-check -i inventory.yml playbook.yml
ansible-playbook -i inventory.yml playbook.yml --limit 10.10.10.111
ansible-playbook -i inventory.yml playbook.yml --limit 10.10.10.111 -e cleanup_execute=true
ansible-playbook -i inventory.yml playbook.yml -e cleanup_execute=true
```

For memory cleanup, use the matching variable:

```bash
ansible-playbook -i inventory.yml cleanup-linux-memory-cache.yml --limit 10.10.10.111 -e memory_cleanup_execute=true
```

If running inside Semaphore, map these concepts to the selected template, inventory, credentials, extra variables, and task log output.

## Anti-Patterns

Do not:

- Assume the local file name is the file Semaphore actually runs.
- Use `hosts: test` unless Semaphore's selected inventory really has a `test` group.
- Treat `gather_facts: false` as a way to avoid Python for normal modules.
- Replace `command` with `shell` to fix target Python errors.
- Use `SUDO="sudo -n"` and expand it as a command on unknown shells.
- Run destructive cleanup by default.
- Run cleanup across all nodes at once before a canary.
- Hide raw command output when a preflight fails.
- Delete container runtime directories manually.
- Claim a cleanup failed solely because of timeout text without verifying final state.

## Output Expectations

When using this skill, report:

- Which playbook path was edited or inspected.
- Whether the task is preview-only or execution-enabled.
- Which inventory/host pattern is expected to match.
- Which target node was used for canary, if any.
- Which validation commands ran and their result.
- Any remaining risk, such as old target Python, missing `crictl.yaml`, runtime timeout, or cache rebuild impact.
