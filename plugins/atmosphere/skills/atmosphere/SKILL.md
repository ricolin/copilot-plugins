---
name: atmosphere
description: "Operating and building Atmosphere environments (vexxhost/atmosphere). Use this skill for operating Atmosphere cloud environments — OpenStack CLI commands (server list, network list, volume list, image list, etc.), Kubernetes/kubectl commands, Ceph storage commands via cephadm, remote shell access to cloud controllers, building all-in-one Atmosphere deployments, and updating AIO environments with specific Ansible tags. Prefers the parallel `atmosphere deploy` / `atmosphere deploy --tags` CLI when available, falling back to `tox -e molecule-aio-*` and `molecule converge` with `ATMOSPHERE_ANSIBLE_TAGS`. Covers nova, neutron, cinder, glance, keystone, heat, octavia, ironic, magnum, kubectl, and cephadm operations."
---

# Atmosphere Environment Operations

This skill is for operating [vexxhost/atmosphere](https://github.com/vexxhost/atmosphere) environments — an open-source platform that deploys OpenStack, Kubernetes, and Ceph on bare metal using Helm and Ansible.

> **Scope:** The deployment, status checking, and validation sections below apply **only to all-in-one (AIO) environments**. Multi-node and production deployments use different procedures not covered here. The [Operating](#operating-an-atmosphere-environment) and [Running Commands](#running-commands) sections apply to any Atmosphere environment.

## Building an All-in-One Environment

Reference: [Atmosphere Quick Start — All-in-One](https://vexxhost.github.io/atmosphere/quick-start.html#all-in-one)

To deploy a complete Atmosphere all-in-one environment on a remote machine.

**System requirements:**
- **OS:** Ubuntu 22.04
- **CPU:** 8 threads minimum (16 recommended for Kubernetes workloads)
- **Memory:** 32GB minimum (64GB recommended for Kubernetes workloads)
- **Nested virtualization:** required if running inside a VM

**Steps:**

1. **SSH to the target host and become root:**

   ```bash
   ssh <USER>@<HOST>    # mode: async; e.g. ubuntu for Ubuntu OS
   sudo -i
   ```

2. **Install dependencies:** Ensure `snapd` is purged, and `git` and `tox` are installed.

   ```bash
   apt-get update
   apt-get install git tox
   ```

3. **Clone and deploy in tmux:**

   The deployment takes a long time. Always run build commands inside a `tmux` session so they persist independently. Launch the build, then immediately return the tmux session name — do not wait for completion.

   **CRITICAL: Always set `remain-on-exit on` so the tmux session stays alive after the command finishes (success or failure). This preserves the full output for later review. Never use the `tmux new -d -s name 'command'` shorthand — it destroys the session on exit.**

   **Preferred command — `atmosphere deploy` (when available):** If the repository checkout provides the `atmosphere` CLI (check for `./bin/atmosphere` in the repo root, or `command -v atmosphere` on `PATH`), **always prefer it** over `tox -e molecule-aio-*`. The `atmosphere` CLI runs roles in parallel and is significantly faster than the molecule path.

   ```bash
   git clone https://github.com/vexxhost/atmosphere
   cd atmosphere
   tmux new -d -s atmosphere \; set-option remain-on-exit on
   # Prefer ./bin/atmosphere deploy when present; fall back to tox otherwise.
   tmux send-keys -t atmosphere "cd $(pwd) && if [ -x ./bin/atmosphere ]; then ./bin/atmosphere deploy -i ./inventory.yaml; else tox -e molecule-aio-ovn; fi" Enter
   ```

   When you already know the CLI is present (e.g. on a host where prior runs used it), invoke it directly:

   ```bash
   tmux send-keys -t atmosphere "cd /root/atmosphere && ./bin/atmosphere deploy -i ./inventory.yaml" Enter
   ```

   This starts the build in the background tmux session `atmosphere` and returns immediately. The session will persist even after the command completes, so output can always be reviewed. Report the session name to track later.

   To deploy from a specific branch or pull request, switch before launching tmux:

   ```bash
   git clone https://github.com/vexxhost/atmosphere
   cd atmosphere
   git checkout <BRANCH>                        # deploy from a branch
   git fetch origin pull/<PR_NUMBER>/head && git checkout FETCH_HEAD  # deploy from a PR
   tmux new -d -s atmosphere \; set-option remain-on-exit on
   tmux send-keys -t atmosphere "cd $(pwd) && if [ -x ./bin/atmosphere ]; then ./bin/atmosphere deploy -i ./inventory.yaml; else tox -e molecule-aio-ovn; fi" Enter
   ```

   **Fallback — `tox -e molecule-aio-*`:** Use only when `./bin/atmosphere` is not present in the checkout. Available molecule scenarios:
   - `molecule-aio-ovn` — all-in-one with OVN networking
   - `molecule-aio-openvswitch` — all-in-one with Open vSwitch networking

   > **OVS vs OVN terminology:** When the user asks for an **"OVS"** or **"Open vSwitch"** build, always use `molecule-aio-openvswitch`. OVS and OVN are **different networking backends** — do not substitute one for the other. "OVS" **never** means OVN. Only use `molecule-aio-ovn` when the user explicitly asks for OVN.

   Set `ATMOSPHERE_DEBUG=true` before running `tox` to enable debug output.

4. **Check deployment progress:**

   See the [Checking All-in-One Deployment Status](#checking-all-in-one-deployment-status) section below for the full procedure. Quick check:

   ```bash
   tmux capture-pane -t atmosphere -p | tail -50
   ```

   This prints the last visible output without attaching interactively. Look for a tox/molecule success or failure summary.

## Checking All-in-One Deployment Status

When asked whether an all-in-one Atmosphere deployment is ready, finished, or about its status, **always check the tmux session on the remote host first**. The AIO deployment runs inside tmux as a long-running `./bin/atmosphere deploy` (preferred) or `tox -e molecule-aio-*` command. The tmux session is the source of truth for whether the build has completed or is still running.

### Procedure

1. **SSH to the remote host:**

   ```bash
   ssh <USER>@<HOST>    # mode: async; e.g. ubuntu for Ubuntu OS
   sudo -i
   ```

2. **List tmux sessions** to find the deployment session:

   ```bash
   tmux ls
   ```

   The deployment session is typically named `atmosphere` (or similar, e.g. `tempest-rerun`, `tempest-verify`).

3. **Capture recent output** from the tmux session to check progress without attaching interactively:

   ```bash
   tmux capture-pane -t atmosphere -p | tail -50
   ```

   This prints the last visible lines from the tmux pane. Look for:
   - **Still running:** ongoing Ansible/Atmosphere/Molecule task output, no final summary yet.
   - **Succeeded:** an `atmosphere deploy` success summary, a Molecule/tox summary line such as `molecule-aio-ovn: commands succeeded` or `congratulations`, or a play recap showing `failed=0`.
   - **Failed:** error messages, a non-zero exit code summary, or `failed=1` (or higher) in the play recap.

4. **Report the result** — tell the user whether the deployment is still running, succeeded, or failed, along with the relevant output lines.

> **Key rule:** Do NOT check Kubernetes pods (`kubectl get pods`) to determine if an AIO deployment is done. The `atmosphere deploy` / tox command orchestrates the full deployment including post-deploy validation. Only the tmux session output tells you the true status.

## Validating an All-in-One Deployment

Deployment and test results are included in the command output in the tmux session on the AIO host.

### Tempest (OpenStack integration tests)

The OpenStack test suite (Tempest) runs as a Kubernetes job `tempest-run-tests` in the `openstack` namespace. Check results with:

```bash
kubectl -n openstack get jobs | grep tempest
kubectl -n openstack logs job/tempest-run-tests
```

### Re-running Tempest tests (AIO only)

To re-run tests, either:

1. **Delete the pod and re-run the deployer in tmux:**

   ```bash
   kubectl -n openstack delete pod -l job-name=tempest-run-tests
   tmux new -d -s tempest-rerun \; set-option remain-on-exit on
   # Prefer ./bin/atmosphere deploy when available; fall back to tox.
   tmux send-keys -t tempest-rerun 'cd /root/atmosphere && if [ -x ./bin/atmosphere ]; then ./bin/atmosphere deploy -i ./inventory.yaml; else tox -e molecule-aio-ovn; fi' Enter
   ```

2. **Run molecule verify directly in tmux** using the existing `.tox` virtualenv:

   ```bash
   tmux new -d -s tempest-verify \; set-option remain-on-exit on
   tmux send-keys -t tempest-verify 'cd /root/atmosphere && source .tox/molecule-aio-ovn/bin/activate && ATMOSPHERE_NETWORK_BACKEND=ovn molecule verify -s aio' Enter
   ```

   Set `ATMOSPHERE_NETWORK_BACKEND` to match the scenario:
   - `ovn` — for OVN deployments
   - `openvswitch` — for Open vSwitch deployments

## Updating an All-in-One Environment with Specific Tags

To apply changes to only specific components of an existing AIO deployment (e.g. just Nova, Neutron, or Keystone), pass `--tags` to the deployer. This runs only the Ansible tasks tagged with the specified components, which is much faster than a full re-deployment. **Always prefer tag-scoped runs over full redeploys** when you only need to update a subset of services.

### Preferred — `atmosphere deploy --tags` (when available)

If `./bin/atmosphere` is present in the checkout, use it with `--tags`. It runs roles in parallel and is significantly faster than `molecule converge`.

1. **SSH to the AIO host and become root:**

   ```bash
   ssh <USER>@<HOST>    # mode: async; e.g. ubuntu for Ubuntu OS
   sudo -i
   ```

2. **Run `atmosphere deploy --tags` in tmux:**

   ```bash
   tmux new -d -s atmosphere-update \; set-option remain-on-exit on
   tmux send-keys -t atmosphere-update 'cd /root/atmosphere && ./bin/atmosphere deploy -i ./inventory.yaml --tags <TAG[,TAG...]>' Enter
   ```

   Replace `<TAG>` with the component(s) to update — comma-separate multiple tags. Examples:

   - `--tags nova` — update only Nova (compute)
   - `--tags neutron` — update only Neutron (networking)
   - `--tags keystone` — update only Keystone (identity)
   - `--tags octavia` — update only Octavia (load balancing)
   - `--tags cinder` — update only Cinder (block storage)
   - `--tags glance` — update only Glance (image service)
   - `--tags heat` — update only Heat (orchestration)
   - `--tags ironic` — update only Ironic (bare metal)
   - `--tags magnum` — update only Magnum (container orchestration)
   - `--tags ironic,nova,neutron,magnum` — update multiple components together

   Pick the **smallest set of tags** that covers what actually changed — this is the main lever for keeping redeploys fast.

3. **Check progress:**

   ```bash
   tmux capture-pane -t atmosphere-update -p | tail -50
   ```

### Fallback — `molecule converge` with `ATMOSPHERE_ANSIBLE_TAGS`

Use this only when `./bin/atmosphere` is not available in the checkout.

1. **SSH to the AIO host and become root:**

   ```bash
   ssh <USER>@<HOST>    # mode: async; e.g. ubuntu for Ubuntu OS
   sudo -i
   ```

2. **Run molecule converge with tags in tmux:**

   ```bash
   tmux new -d -s atmosphere-update \; set-option remain-on-exit on
   tmux send-keys -t atmosphere-update 'cd /root/atmosphere && source .tox/molecule-aio-ovn/bin/activate && ATMOSPHERE_ANSIBLE_TAGS=<TAG[,TAG...]> molecule converge -s aio' Enter
   ```

   Same tag values as the `atmosphere deploy --tags` list above (comma-separated for multiples, e.g. `ATMOSPHERE_ANSIBLE_TAGS=nova,neutron`).

   For Open vSwitch deployments, activate the corresponding virtualenv instead:

   ```bash
   source .tox/molecule-aio-openvswitch/bin/activate
   ```

3. **Check progress:**

   ```bash
   tmux capture-pane -t atmosphere-update -p | tail -50
   ```

> **Key rule:** The `.tox/molecule-aio-*/bin/activate` virtualenv must already exist from a previous full `tox -e molecule-aio-*` run. If it does not exist, run a full deployment first (or use `./bin/atmosphere deploy` if available).

### Changing Molecule AIO Configuration Variables

To customize the AIO deployment (e.g. override Ansible variables, feature flags, or component settings), edit the molecule variable files **before** running `molecule converge`.

**Where to make changes (in priority order):**

1. **`molecule/default/group_vars/all/molecule.yml`** — the primary group variables file. Edit here first for variables that apply across all scenarios.
2. **`molecule/aio/`** — if the variable is specific to the AIO scenario or not picked up from `molecule/default/group_vars/`, edit the relevant files here (e.g. `converge.yml`, `prepare.yml`, `molecule.yml`).

> **CRITICAL:** When setting `ATMOSPHERE_ANSIBLE_VARS_PATH` for AIO deployments, it **must** point to the **scenario directory** (`molecule/aio/`), **not** `molecule/default/`. AIO vars are scenario-specific and live under `molecule/aio/`. Pointing to the wrong directory will cause variables to be missing or incorrect.
>
> ```bash
> # Correct — points to the AIO scenario directory:
> ATMOSPHERE_ANSIBLE_VARS_PATH=/root/atmosphere/molecule/aio/
>
> # Wrong — molecule/default/ does not contain AIO-specific vars:
> ATMOSPHERE_ANSIBLE_VARS_PATH=/root/atmosphere/molecule/default/
> ```

## Operating an Atmosphere Environment

- **User:** OS-dependent (e.g. `ubuntu` for Ubuntu, `centos` for CentOS, `root` for some environments)
- **OpenStack credentials:** source `/root/openrc` on any control plane node before running OpenStack CLI commands

## Running Commands

Use an interactive SSH session. First SSH to the host, then **always `sudo -i` to become root before running any commands**. `openstack`, `kubectl`, and `cephadm` all require root. After becoming root, source the OpenStack credentials with `source /root/openrc`.

### Procedure

1. **Start an async SSH session:**

   ```bash
   ssh <USER>@<HOST>    # mode: async; e.g. ubuntu for Ubuntu OS
   ```

2. **Become root and source credentials:**

   ```
   sudo -i
   source /root/openrc
   ```

3. **Run commands** — OpenStack, kubectl, and cephadm are now available:

   ```
   openstack flavor list
   openstack server list
   kubectl get nodes
   cephadm shell -- ceph status
   ```

**Never run `openstack`, `kubectl`, or `cephadm` without becoming root first — always `sudo -i`.**

### OpenStack command examples

```
openstack server list
openstack server show <NAME_OR_ID>
openstack network list
openstack volume list
openstack image list
openstack flavor list
```

### Kubernetes command examples

```
kubectl get pods -A
kubectl get nodes
kubectl describe pod <POD> -n <NAMESPACE>
kubectl logs <POD> -n <NAMESPACE>
```

### Ceph command examples

Ceph is managed via `cephadm`. Use `cephadm shell` to enter the Ceph container, or pass commands directly:

```
cephadm shell -- ceph status
cephadm shell -- ceph osd tree
cephadm shell -- ceph df
cephadm shell -- ceph health detail
cephadm shell -- rados df
cephadm shell -- ceph osd pool ls detail
```

### Horizon Dashboard

To get the dashboard URL:

```
kubectl -n openstack get ingress/dashboard -ojsonpath='{.spec.rules[0].host}'
```

Login credentials can be found in `/root/openrc` (`OS_USERNAME`, `OS_PASSWORD`, `OS_USER_DOMAIN_NAME`).

## tmux Session Lifecycle Rules

**These rules apply to all tmux sessions created for deployment or validation tasks.**

1. **Always use `remain-on-exit on`** when creating tmux sessions. This keeps the session alive after the command finishes so output can be reviewed. Create sessions with:
   ```bash
   tmux new -d -s <SESSION_NAME> \; set-option remain-on-exit on
   tmux send-keys -t <SESSION_NAME> '<COMMAND>' Enter
   ```
   Never use `tmux new -d -s <name> '<command>'` — that form destroys the session when the command exits.

2. **Never kill, close, or stop a tmux session** while a deployment or validation task is still running inside it. Always check if the task has finished before cleaning up:
   ```bash
   tmux ls                                    # check if session exists
   tmux capture-pane -t <SESSION_NAME> -p | tail -50  # check output for completion
   ```

3. **Only clean up after confirming completion.** Once you have confirmed the deployment or validation has finished (succeeded or failed) and the user has been informed, sessions can be cleaned up with:
   ```bash
   tmux kill-session -t <SESSION_NAME>
   ```

4. **If `tmux ls` shows no sessions**, the deployment was either not started in tmux or the session was lost. Do not assume the deployment is done — investigate further by checking running processes and pod status.

## Deployment Debugging

When an AIO deployment fails, check the tmux session output for the error. Below are known failure scenarios and their fixes. After applying a fix, **always re-run the deployment** using the same tmux procedure described in [Building an All-in-One Environment](#building-an-all-in-one-environment).

### "There are existing loopback devices"

**Symptom:** The deployment fails on the Ansible task `Fail if there is any existing loopback devices` with a message about existing loopback devices.

**Root cause:** The `create_fake_devices.yml` playbook has a safety check that aborts if loopback devices already exist (e.g. from a previous failed deployment attempt).

**Fix:** Remove the failing task directly from the playbook on the remote host, then re-run the deployment.

```bash
# On the remote host as root, remove the "Fail if there is any existing loopback devices" task
# from the installed Ansible collection playbook:
PLAYBOOK=~/.ansible/collections/ansible_collections/vexxhost/ceph/playbooks/create_fake_devices.yml
# Use sed or Python to remove the task block that contains "Fail if there is any existing loopback devices"
python3 -c "
import yaml, sys
with open('$PLAYBOOK') as f:
    data = yaml.safe_load(f)
for play in data:
    if 'tasks' in play:
        play['tasks'] = [t for t in play['tasks'] if t.get('name') != 'Fail if there is any existing loopback devices']
with open('$PLAYBOOK', 'w') as f:
    yaml.dump(data, f, default_flow_style=False, sort_keys=False)
print('Removed the loopback device check task')
"
```

After removing the task, re-run the deployment in a new tmux session:

```bash
tmux kill-session -t atmosphere 2>/dev/null
tmux new -d -s atmosphere \; set-option remain-on-exit on
tmux send-keys -t atmosphere "cd /root/atmosphere && if [ -x ./bin/atmosphere ]; then ./bin/atmosphere deploy -i ./inventory.yaml; else tox -e molecule-aio-ovn; fi" Enter
```

## Important

- Use an **async interactive SSH session** — start with `ssh` (mode: async), then send `sudo -i` and commands via `write_bash`.
- For commands that produce wide output, append `--format json` (OpenStack) or `-o json` (kubectl) for machine-readable results.
- If a command requires multiple steps, chain them with `&&`.
- When done, exit the session with `exit` twice (once for root, once for the SSH user).

