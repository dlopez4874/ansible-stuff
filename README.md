# ansible-stuff

Ansible control repo for a small Arch Linux k3s lab cluster running on XCP-ng.
Three nodes, one server and two agents, managed from a Kubuntu control node.

Everything here is run ad-hoc from the control node — no CI, no roles, just
playbooks kept deliberately flat and readable.

## Playbooks

| Playbook | What it does |
|---|---|
| `k3s_install.yml` | Prepare kernel/sysctl and tooling, install the k3s server |
| `k3s_workers_install.yml` | Install/enforce the k3s agent on workers, align runtime version |
| `k3s_fetch_kubeconfig.yml` | Pull kubeconfig from the server to the control node |
| `k3s_rolling_reboot.yml` | Drain, reboot and uncordon workers one at a time (with delays for XCP-ng's network drop across reboot) |
| `deploy_nginx.yml` | Deploy nginx via the Kubernetes API |
| `ingress_nginx.yml` | Create a Traefik Ingress for nginx |
| `archupdate.yml` | `pacman -Syu` across the cluster, then prune the package cache |
| `arch-split-dns.yml` | Runtime-only split DNS via systemd-resolved (routes one domain at the AD DNS, leaves DHCP DNS alone) |
| `kerb-cifs.yml` | Persistent Kerberos CIFS mount of a FlashArray file share (fstab + systemd automount) |
| `kerb-cifs-ephemeral.yml` | Same mount, one-shot, no fstab changes |
| `unmount-cifs.yml` | Unmount the CIFS share |
| `aur_audit.yml` | Audit nodes for indicators of compromise from the 2026 AUR supply-chain attacks — see below |
| `fa-get-fqdn.yml` | Smoke-test the `everpure.flasharray` collection against a lab FlashArray |

### aur_audit.yml

The largest thing here. Sixteen checks across two campaigns with different
delivery shapes: the June 2026 "Atomic Arch" waves (PKGBUILD → npm/bun → Rust
infostealer, obfuscated shell, shell-config injection) and the July–August 2026
account-takeover wave (a precompiled ELF committed into the package's own AUR
git tree and called from `build()`).

Detection notes worth knowing:

- The committed-ELF check uses `git ls-files`, not `find -name`. yay/paru cache
  dirs are git clones of the AUR repo, so an attacker-committed blob is *tracked*
  while ordinary build output is not. A blunt "any ELF under the cache" scan
  produces 20+ false positives per node from one package's build tree alone.
- Install-date windows are data (`ioc_windows`), not hard-coded month regexes,
  so adding a campaign needs no code change.
- It cross-references the community IOC campaign index at runtime and reports
  explicitly when that index does *not* cover the currently open window, rather
  than implying coverage it doesn't have.

Reports land in `aur_audit_results/` on the control node (gitignored).

## Layout

```
ansible.cfg          inventory path, become settings, ssh args
inventory            host groups: arch, k3s_server, k3s_agents   (gitignored)
group_vars/all.yml   secrets, ansible-vault encrypted            (gitignored)
group_vars/arch.yml  lab-specific values: domains, IPs, shares    (gitignored)
host_vars/           per-host overrides                          (gitignored)
venv.readme          venv + collection setup notes
```

## Secrets and lab-specific values

Nothing environment-specific is committed. This repo is public, so:

- **Secrets** live in `group_vars/all.yml`, ansible-vault encrypted, unlocked
  with `--vault-password-file .secret`. Both `.secret` and the vault file are
  gitignored.
- **Non-secret but internal values** (AD domain and realm, DNS and k3s server
  IPs, SMB server and share names) live in `group_vars/arch.yml`, gitignored.
  The playbooks reference them only by variable name — `ad_domain`, `ad_realm`,
  `ad_dns`, `smb_server`, `smb_share`, `k3s_server_url`.
- `*.local.readme` is gitignored and used for local-only notes about where
  values live.

A fresh clone therefore won't run as-is: populate `inventory`, `group_vars/` and
`.secret` first. That's intentional — the alternative is tracked files needing
permanent local edits, which is how a working copy silently drifts from `HEAD`.

## Usage

Setup is in `venv.readme`. Two aliases do the work:

```bash
alias adhoc="ansible --vault-password-file .secret"
alias playbook="ansible-playbook --vault-password-file .secret"
```

```bash
playbook aur_audit.yml                    # full audit, reports fetched locally
playbook aur_audit.yml --tags fetch       # re-fetch reports only
playbook k3s_rolling_reboot.yml
adhoc arch -m ping
```

Edit secrets with:

```bash
ansible-vault edit --vault-password-file .secret group_vars/all.yml
```

## License

See `LICENSE`.
