---
name: atmosphere
description: "Operating and building Atmosphere environments (vexxhost/atmosphere). Use this skill for operating Atmosphere cloud environments — OpenStack CLI commands (server list, network list, volume list, image list, etc.), Kubernetes/kubectl commands, Ceph storage commands via cephadm, remote shell access to cloud controllers, and building all-in-one Atmosphere deployments. Covers nova, neutron, cinder, glance, keystone, heat, octavia, kubectl, and cephadm operations."
---

# Atmosphere Environment Operations

This skill is for operating [vexxhost/atmosphere](https://github.com/vexxhost/atmosphere) environments — an open-source platform that deploys OpenStack, Kubernetes, and Ceph on bare metal using Helm and Ansible.

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

   ```bash
   git clone https://github.com/vexxhost/atmosphere
   cd atmosphere
   tmux new -d -s atmosphere 'cd $(pwd) && tox -e molecule-aio-ovn'
   ```

   This starts the build in the background tmux session `atmosphere` and returns immediately. Report the session name to track later.

   To deploy from a specific branch or pull request, switch before launching tmux:

   ```bash
   git clone https://github.com/vexxhost/atmosphere
   cd atmosphere
   git checkout <BRANCH>                        # deploy from a branch
   git fetch origin pull/<PR_NUMBER>/head && git checkout FETCH_HEAD  # deploy from a PR
   tmux new -d -s atmosphere 'cd $(pwd) && tox -e molecule-aio-ovn'
   ```

   Available molecule scenarios:
   - `molecule-aio-ovn` — all-in-one with OVN networking
   - `molecule-aio-openvswitch` — all-in-one with Open vSwitch networking

   Set `ATMOSPHERE_DEBUG=true` before running `tox` to enable debug output.

4. **Check deployment progress:**

   Attach to the tmux session to see if the command has finished:

   ```bash
   tmux attach -t atmosphere
   ```

   If the `tox` command completed successfully, the session will show the final output. Detach again with `Ctrl+b d`.

## Validating a Deployment

Deployment and test results are included in the command output in tmux.

### Tempest (OpenStack integration tests)

The OpenStack test suite (Tempest) runs as a Kubernetes job `tempest-run-tests` in the `openstack` namespace. Check results with:

```bash
kubectl -n openstack get jobs | grep tempest
kubectl -n openstack logs job/tempest-run-tests
```

### Re-running Tempest tests

To re-run tests, either:

1. **Delete the pod and re-run tox in tmux:**

   ```bash
   kubectl -n openstack delete pod -l job-name=tempest-run-tests
   tmux new -d -s tempest-rerun 'cd /root/atmosphere && tox -e molecule-aio-ovn'
   ```

2. **Run molecule verify directly in tmux** using the existing `.tox` virtualenv:

   ```bash
   tmux new -d -s tempest-verify 'cd /root/atmosphere && source .tox/molecule-aio-ovn/bin/activate && ATMOSPHERE_NETWORK_BACKEND=ovn molecule verify -s aio'
   ```

   Set `ATMOSPHERE_NETWORK_BACKEND` to match the scenario:
   - `ovn` — for OVN deployments
   - `openvswitch` — for Open vSwitch deployments

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

## Important

- Use an **async interactive SSH session** — start with `ssh` (mode: async), then send `sudo -i` and commands via `write_bash`.
- For commands that produce wide output, append `--format json` (OpenStack) or `-o json` (kubectl) for machine-readable results.
- If a command requires multiple steps, chain them with `&&`.
- When done, exit the session with `exit` twice (once for root, once for the SSH user).

