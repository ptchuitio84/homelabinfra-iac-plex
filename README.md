# plex-ansible

Ansible automation for deploying and configuring Plex Media Server on the NNT homelab media node. Handles everything from LVM storage setup and OS configuration to Plex installation, Nginx reverse proxy, SELinux, and firewall rules — idempotent, Jenkins-driven, dry-run by default.

---

## Architecture

```
Jenkins pipeline (nnt-jkn-plex-deploy)
    │
    ▼ runs on hmvlapans001 (Ansible control node)
    │
    ├── Lint + Syntax check (ansible-lint)
    │
    └── Deploy → hmpplxap002 (Plex media server)
                    │
                    ├── User/group provisioning (uid 12365, gid 5041)
                    ├── Repo setup (Plex + Nginx GPG-signed repos)
                    ├── Package install (plexmediaserver, nginx)
                    ├── LVM: /dev/sdb + /dev/sdc → vg_plex → lv_plex
                    ├── Filesystem + mount at /plex (ext4)
                    ├── Media library directories (/plex/Library/...)
                    ├── Firewall rules (32400 + discovery ports)
                    ├── Nginx reverse proxy config
                    ├── SELinux config
                    └── Plex Preferences.xml restore
```

**Target host:** `hmpplxap002` (10.100.7.10) — CentOS 7, bare-metal or VM, root SSH access  
**Media storage:** LVM across two physical disks (`/dev/sdb`, `/dev/sdc`)  
**Network access:** Plex UI at port 32400; Nginx reverse proxy at 80/443  

---

## Repository Structure

```
plex-ansible/
├── site.yml                          # Top-level playbook → plex role
├── Jenkinsfile                        # CI/CD pipeline definition
├── inventory/
│   └── hosts.yml                      # hmpplxap002 (10.100.7.10)
├── group_vars/
│   └── plex_servers.yml               # Group-level vars (if present)
└── roles/
    └── plex/
        ├── defaults/main.yml           # All tunable variables
        ├── handlers/main.yml           # firewalld reload, nginx restart
        ├── tasks/
        │   ├── main.yml               # Imports sub-tasks in order
        │   ├── user.yml               # Plex user/group
        │   ├── repos.yml              # Plex + Nginx yum repos
        │   ├── install.yml            # Package install + service enable
        │   ├── lvm.yml                # Volume group, logical volume, filesystem
        │   ├── directories.yml        # Media library directory tree
        │   ├── firewall.yml           # Port rules (TCP + UDP)
        │   ├── nginx.yml              # Reverse proxy config
        │   ├── selinux.yml            # SELinux policy
        │   └── config.yml             # Preferences.xml + DB snapshot restore
        ├── templates/
        │   └── plex.conf.j2           # Plex config Jinja2 template
        └── files/
            ├── Preferences.xml        # Plex preferences backup
            ├── plex.conf.orig         # Original Plex config backup
            └── Databases/             # Plex database snapshots
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Target OS | CentOS 7 (yum, firewalld, SELinux) |
| Disks | `/dev/sdb` and `/dev/sdc` — unpartitioned, used for LVM |
| Network | Outbound internet access (Plex + Nginx repos) |
| SSH | Root access from Ansible control node (`hmvlapans001`) |
| Ansible | ansible-core 2.15+ on `hmvlapans001` |

**Disk warning:** The LVM task (`tasks/lvm.yml`) will format `/dev/sdb` and `/dev/sdc`. If those devices already have data, it will be destroyed. Verify device names match before running.

---

## Jenkins Pipeline

**Pipeline:** `nnt-jkn-plex-deploy`  
**Agent:** `ans001` (hmvlapans001)  
**Jenkinsfile:** `Jenkinsfile` (this repo, root)

Parameters:

| Parameter | Default | Description |
|---|---|---|
| `TARGET_HOST` | `plex_servers` | Ansible limit — target host or group |
| `DRY_RUN` | `true` | Run with `--check` flag (no changes applied) |

Stages:
1. **Checkout** — clone this repo
2. **Lint** — `ansible-lint site.yml`
3. **Syntax Check** — `ansible-playbook --syntax-check`
4. **Deploy** — `ansible-playbook site.yml [--check if DRY_RUN]`

Always run with `DRY_RUN=true` first to validate what will change before committing to a live deploy.

---

## Manual Run (from hmvlapans001)

```bash
cd /opt/plex-ansible

# Dry run — shows planned changes, applies nothing
ansible-playbook site.yml --check --limit hmpplxap002

# Live deploy
ansible-playbook site.yml --limit hmpplxap002

# Target a specific role task (e.g., firewall only)
ansible-playbook site.yml --limit hmpplxap002 --tags firewall
```

---

## Storage Layout

Plex media lives on a dedicated LVM volume:

```
/dev/sdb  ─┐
            ├── VG: vg_plex → LV: lv_plex → /plex (ext4)
/dev/sdc  ─┘
```

Media directories under `/plex`:

```
/plex/
└── Library/
    └── Application Support/Plex Media Server/
        ├── Media/         ← raw media files
        ├── Metadata/      ← scraped metadata cache
        └── Plug-in Support/Databases/  ← Plex SQLite databases
```

The 13 media library categories (Adventure, Animation, Anime, Disney, Documentary, etc.) are defined in `roles/plex/defaults/main.yml` under `plex_libraries`. Add or remove libraries there before the next run.

---

## Firewall Rules Applied

| Protocol | Ports | Purpose |
|---|---|---|
| TCP | 80, 443 | Nginx reverse proxy |
| TCP | 32400 | Plex Media Server (direct access) |
| TCP | 3005, 8324, 32469 | Plex Companion, Roku, DLNA |
| UDP | 5353, 1900 | mDNS, SSDP (discovery) |
| UDP | 32410–32414 | GDM network discovery |

---

## Configuration Restore

The `config.yml` task restores a known-good Plex configuration from files checked into this repo:

- `files/Preferences.xml` — Plex server preferences (server name, claim token, library locations)
- `files/Databases/*.db` — Plex library database snapshots

**When to update these files:** After any manual configuration change in the Plex UI that you want to persist across rebuilds, export the relevant files from `/plex/Library/Application Support/Plex Media Server/` and commit them here.

This is a manual snapshot process — not automated backup. For automated backups, a separate playbook would be needed.

---

## Customization

All tunable parameters are in `roles/plex/defaults/main.yml`:

| Variable | Description |
|---|---|
| `plex_user` / `plex_uid` | Plex service account (uid 12365) |
| `plex_group` / `plex_gid` | Plex group (gid 5041) |
| `plex_mount_path` | Storage mount point (`/plex`) |
| `plex_vg_name` / `plex_lv_name` | LVM volume group and logical volume names |
| `plex_disks` | List of disks for the volume group |
| `plex_libraries` | List of media library directory names |
| `plex_ports_tcp` / `plex_ports_udp` | Firewall port lists |

---

## Known Limitations

- **Single host only** — inventory and defaults assume one Plex server. Multi-instance requires refactoring.
- **CentOS 7 (EOL June 2024)** — target OS is end-of-life. Upgrade path is OL9 (Oracle Linux 9), which requires Plex and Nginx repo URL updates and likely SELinux policy changes.
- **Root SSH** — `ansible_user=root` is overly permissive. Should be replaced with a sudo-enabled service account.
- **No automated backup** — database snapshots are manual; no scheduled export or retention policy.
- **Hardcoded library list** — adding new media libraries requires editing defaults and re-running the playbook.
