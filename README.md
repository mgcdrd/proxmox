# mgcdrd.proxmox

Proxmox VE management over the API, built on `community.proxmox` (>= 2.1.0).
Every role runs against the PVE API from the controller (`delegate_to: localhost`);
none of them need SSH to the hypervisor. The exception is `proxmox_disk`, which
also looks inside the guest for the new block device.

Role variables are in each role's README (`roles/<role>/README.md`); the whole
set is summarised in [PROXMOX_VARIABLES.md](PROXMOX_VARIABLES.md).

## Roles

### Guests

| Role | Description |
|---|---|
| `proxmox_vm` | Clones, lifecycle-manages, and templates VMs via the ProxMox API; queries cluster/node/storage info |
| `proxmox_lxc` | Creates, clones, lifecycle-manages, and templates LXC containers via the ProxMox API |
| `proxmox_disk` | Adds or resizes a disk on a ProxMox VM via the PVE API, then detects the resulting block device inside the VM |
| `proxmox_nic` | Adds, updates, and removes ProxMox VM network interfaces (net0-net31) via the PVE API |
| `proxmox_snapshot` | Creates, deletes, and rolls back ProxMox VE VM/CT snapshots via the PVE API |
| `proxmox_guest_exec` | Runs a command inside a VM through the QEMU guest agent, via the PVE API (no SSH); token auth only |
| `proxmox_vm_manage` | Applies disk, NIC, and snapshot changes to one VM by calling `proxmox_disk`, `proxmox_nic`, and `proxmox_snapshot` behind one variable interface |

### Cluster and platform

| Role | Description |
|---|---|
| `proxmox_node` | Manages ProxMox VE node power state, certificates/DNS/subscription config, and task monitoring via the PVE API |
| `proxmox_cluster` | Creates a ProxMox VE cluster and joins additional nodes to it via the PVE API |
| `proxmox_network` | Manages node-level physical network interfaces (bridges/bonds/VLANs/OVS) via the PVE API |
| `proxmox_sdn` | Manages ProxMox VE Software-Defined Networking — zones, vnets, and subnets — via the PVE API |
| `proxmox_storage` | Creates/deletes ProxMox VE storage backend definitions (NFS/CIFS/dir/iSCSI/CephFS/RBD/PBS/ZFS) via the PVE API |
| `proxmox_firewall` | Manages ProxMox VE's built-in firewall — options, rules, security groups, aliases, IP sets — via the PVE API |
| `proxmox_access` | Manages ProxMox VE users, groups, IAM roles, ACLs, and auth realms via the PVE API |
| `proxmox_acme` | Manages ACME certificates for the PVE node's own web UI (pveproxy) via the PVE API |
| `proxmox_ceph` | Manages ProxMox VE-integrated Ceph monitors/managers/MDS, OSDs, and pools via the PVE API |
| `proxmox_ha` | Manages ProxMox VE High Availability groups, resources, and node/resource-affinity rules via the PVE API |
| `proxmox_pool` | Creates/deletes ProxMox VE resource pools and manages VM/storage pool membership via the PVE API |
| `proxmox_backup` | Manages ProxMox VE backup jobs, VM job membership, and on-demand backups via the PVE API |

## Installation

Add to `collections/requirements.yml`. List `community.proxmox` separately; this
collection deliberately has no `dependencies:` block in `galaxy.yml`.

```yaml
collections:
  - name: mgcdrd.proxmox
    source: https://github.com/mgcdrd/proxmox.git
    type: git
    version: v0.1.1

  - name: community.proxmox
    version: ">=2.1.0"
```

Requires Ansible core >= 2.17 and the `proxmoxer` and `requests` Python libraries on the controller.

## Shared connection variables

Every `proxmox_*` role talks to the PVE API through the exact same six
variables — there's no role-prefixed variant of these anymore:

```yaml
proxmox_api_host: "pve2.example.com"   # any PVE node — it manages the whole cluster
proxmox_api_user: "root@pam"
proxmox_api_password: "{{ vault_proxmox_api_password }}"
# or, recommended:
proxmox_api_token_id: "{{ vault_proxmox_api_token_id }}"
proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
proxmox_validate_certs: false
```

| Variable | Default | Required | Description |
|---|---|---|---|
| `proxmox_api_host` | none | Yes | Any PVE node's FQDN; it manages the whole cluster |
| `proxmox_api_user` | none | Yes | API user, e.g. `root@pam` |
| `proxmox_validate_certs` | none | Yes | Validate the PVE API TLS certificate |
| `proxmox_api_password` | none | One auth method | Password for `proxmox_api_user`; use `{{ vault_proxmox_api_password }}` |
| `proxmox_api_token_id` | none | One auth method | API token ID, recommended over a password; use a `vault_` variable |
| `proxmox_api_token_secret` | none | With `proxmox_api_token_id` | API token secret; use a `vault_` variable |
| `proxmox_acme_api_password` | none | For `proxmox_acme` | `root@pam` password used only by `proxmox_acme`, which the PVE API restricts to password auth |
| `proxmox_target_vm_name` | `inventory_hostname` | No | VM that `proxmox_disk`, `proxmox_nic`, `proxmox_vm_manage`, and `proxmox_guest_exec` act on; each role has its own `proxmox_<role>_vm_name` that defaults from it |

A VM/CT lives on one node in one cluster — it never has a different API
endpoint or credential depending on which `proxmox_*` role happens to be
acting on it, so there's nothing to override per role. Set these once
(deployment `group_vars`/`vault.yml`, or `inventory-common` group_vars for
a shared PVE cluster group) and every role in a play picks them up.
Multiple PVE clusters are an inventory concern — a different host group
with its own `proxmox_api_host` etc., not a per-role variable — see
`inventory-common/README.md`.

Each role passes these to the `community.proxmox` modules once, through a
`module_defaults` block on the `community.proxmox.proxmox` action group in
its `tasks/main.yml`, so the individual tasks don't repeat them. This needs
`community.proxmox` >= 2.1.0. A task can still set its own `api_*` argument
to override the default (`proxmox_cluster` join does this per node).
`ansible.builtin.uri` tasks (`proxmox_backup` jobs, `proxmox_guest_exec`)
aren't covered by the action group and read `proxmox_api_*` directly.

None of these vars have a collection-level default (the same way
`vault_proxmox_api_password` never has). `proxmox_api_host`/`proxmox_api_user`/
`proxmox_validate_certs` fail loudly with an "undefined variable" error if
unset — there's no sensible default for an endpoint/cluster identity, and
guessing wrong would mean silently talking to the wrong cluster.
`proxmox_api_password`/`proxmox_api_token_id`/`proxmox_api_token_secret`
are deliberately different: pick **one** auth method and leave the other
pair unset — every task wraps them in `| default(omit)`, so the unused
pair is simply never sent to the module rather than causing a crash or a
`required_one_of` error demanding you also populate a method you don't
want.

**One documented exception:** `proxmox_acme` (ACME account/plugin/certificate
management) is a hard PVE API restriction to `root@pam` password auth —
tokens are rejected outright. It does not use the shared
`proxmox_api_password`/`proxmox_api_token_id` for those tasks; it has its
own `proxmox_acme_api_password`. If it shared the connection vars, setting
a password to satisfy `proxmox_acme` would silently switch every other
`proxmox_*` role in the same play from token auth to password auth too
(the underlying module always prefers password over token when both are
present, with no way to force token). See that role's README.

`proxmox_disk`/`proxmox_nic`/`proxmox_vm_manage` also target a specific VM
by name — that one's legitimately per-invocation (a play can manage
several different VMs), so it stays a role-prefixed var defaulting from
the shared `proxmox_target_vm_name` — see each role's own README for
"VM targeting".
