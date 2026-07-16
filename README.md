# rocky10-pg-base

Base Ansible role for turning a fresh Rocky Linux 10 VM into a hardened
PostgreSQL 18 + pgBackRest node. Meant to be your "golden" role that every
future DB VM (Proxmox clone or fresh install) runs against.

## Layout

```
ansible-rocky-pg-base/
├── playbook.yml                  # entry point, applies the role to db_servers group
├── inventory/hosts.example.ini   # copy/rename, or build one dynamically in Semaphore
├── group_vars/db_servers.yml     # per-environment overrides (vars below)
└── roles/rocky10_pg_base/
    ├── defaults/main.yml         # all tunables live here, override in group_vars/host_vars
    ├── handlers/main.yml
    ├── tasks/
    │   ├── main.yml              # imports the others in order
    │   ├── hardening.yml         # SSH, firewalld, updates, fail2ban, chrony
    │   ├── cockpit.yml           # disables the web console on :9090
    │   ├── postgresql.yml        # PGDG repo, install, initdb, postgresql.conf, pg_hba.conf
    │   └── pgbackrest.yml        # install + pgbackrest.conf + stanza create + backup timer
    └── templates/
        ├── postgresql.conf.j2
        ├── pg_hba.conf.j2
        └── pgbackrest.conf.j2
```

## Using this with SemaphoreUI

1. **Project** → create one project, e.g. "Database Fleet".
2. **Key Store** → add an SSH key credential (the key Semaphore will use to
   reach the VMs) and, separately, a "Login with password" credential if you
   ever need `become` with a password instead of NOPASSWD sudo.
3. **Repository** → point Semaphore at the git repo containing this
   directory (push it to your own git server/GitHub first — Semaphore
   pulls from git, it doesn't accept raw file uploads for playbooks).
4. **Inventory** → create a Static or file-based inventory in Semaphore that
   lists your DB VMs (or point it at `inventory/hosts.example.ini` in the
   repo and let Semaphore render it). Attach the SSH key credential here.
5. **Environment** (optional but useful) → this is Semaphore's place for
   extra vars / secrets (pgBackRest repo passphrase, DB passwords) so they
   don't live in group_vars in plaintext. Reference them in the playbook run
   as `--extra-vars @environment` (Semaphore does this automatically when
   you attach an Environment to the template).
6. **Task Template** → New Template → Playbook `playbook.yml`, pick the
   Repository, Inventory, SSH key, and Environment from the dropdowns.
   Run it. Every future DB VM = clone the Proxmox template, add it to
   inventory, hit Run.
7. Once this works, add a **Schedule** on a second template that just runs
   `pgbackrest --stanza=main backup` on a cron (see the systemd timer note
   in `tasks/pgbackrest.yml` — you can automate backups either at the OS
   level via the timer this role installs, or centrally from Semaphore's
   scheduler; pick one, not both, to avoid double backups).

## Proxmox → Rocky 10 → this role, end to end

1. On the Proxmox host, build (once) a Rocky Linux 10 cloud-init template:
   download the Rocky 10 GenericCloud qcow2, `qm create` a VM, `qm importdisk`
   the image in, attach it, add a `cloudinit` drive, set `qm set` for
   ciuser/sshkey/ipconfig, then `qm template` it.
2. For each new DB VM: `qm clone <template-id> <new-id> --name pg-01 --full`,
   set per-VM IP via `qm set <new-id> --ipconfig0 ip=10.x.x.x/24,gw=10.x.x.1`,
   start it.
3. Add the VM's IP/hostname to your Semaphore inventory.
4. Run the `playbook.yml` template in Semaphore against just that host
   (`--limit pg-01`).
5. VM now has: hardened SSH/firewall, Cockpit disabled, PostgreSQL 18
   running and tuned from `postgresql.conf.j2`, and pgBackRest configured
   and stanza-created.

See inline comments in each task file for what to change per-environment.
