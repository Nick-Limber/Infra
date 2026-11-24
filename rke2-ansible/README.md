# RKE2 with Ansible

Two playbooks: one builds the control plane, the other bootstraps workers.

```
control-plane.yml   install rke2-server (first host bootstraps, extras join serially)
workers.yml         install rke2-agent and join workers (re-runnable for new nodes)
site.yml            both, in order
reset.yml           DESTRUCTIVE uninstall
group_vars/all.yml  all settings (version, token, CNI, CIDRs, taints...)
inventory/hosts.ini your hosts
```

## Requirements

- Ansible 2.15+ on your workstation
- Targets: systemd-based Linux (Ubuntu/Debian, RHEL/Rocky/Alma, SLES), SSH + sudo,
  outbound internet access to `get.rke2.io` and GitHub releases
- Network ports open between nodes: TCP 9345, 6443, 10250, 2379-2380 (servers),
  UDP 8472 (canal VXLAN), TCP 9099, NodePorts 30000-32767

```bash
ansible-galaxy collection install -r requirements.yml
```

## Usage

1. Edit `inventory/hosts.ini` with your IPs and SSH user.
2. Set a token in `group_vars/all.yml` (recommended), ideally vaulted:
   ```bash
   ansible-vault encrypt_string "$(openssl rand -hex 32)" --name rke2_token
   ```
   If left empty, the token RKE2 generates on the first server is read and used automatically.
3. Optionally pin `rke2_version` (e.g. `v1.33.5+rke2r1`) instead of the `stable` channel.
4. Run:
   ```bash
   ansible all -m ping                 # sanity check
   ansible-playbook control-plane.yml
   ansible-playbook workers.yml
   ```
5. Use the cluster:
   ```bash
   export KUBECONFIG=$PWD/kubeconfig/rke2.yaml
   kubectl get nodes -o wide
   ```

## Adding a worker later

Add it under `[rke2_agents]` and run `ansible-playbook workers.yml --limit <newhost>`.

## HA control plane

Add more hosts (3 total) under `[rke2_servers]`. Put a load balancer/VIP in front of
ports 9345 and 6443 and set `rke2_api_endpoint` to it. Add its DNS name to `rke2_tls_san`.

## Upgrades

Change `rke2_version`, then re-run with `-e rke2_force_install=true`. The installer
re-runs and the service restarts. Servers go one at a time automatically after the
first; for workers, add `-e` plus `--forks 1` (or `--limit`) to roll them gradually:

```bash
ansible-playbook control-plane.yml -e rke2_force_install=true
ansible-playbook workers.yml -e rke2_force_install=true --forks 1
```

## Notes

- Control plane nodes are tainted `CriticalAddonsOnly=true:NoExecute` so regular
  workloads only run on workers. Set `rke2_control_plane_taint: false` to allow it.
- The default kubeconfig mode is `0600` (root only). `kubectl` is symlinked to
  `/usr/local/bin` on servers; use `sudo kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml ...`.
- Logs: `journalctl -u rke2-server -f` / `journalctl -u rke2-agent -f`.
