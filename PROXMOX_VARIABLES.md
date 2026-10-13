Master Variables List for the proxmox collection
====================================================

Every `proxmox_*` role talks to the PVE API through `community.proxmox`
(>= 2.1.0). Connection variables are shared by all of them and set once
(see **Shared connection variables**). Each role is then split into
independent sections, switched on with `<role>_do_<section>` gates and run
by tag, so one role can be applied piecemeal.

Guests: [vm](#proxmox_vm) · [lxc](#proxmox_lxc) · [disk](#proxmox_disk) · [nic](#proxmox_nic) · [snapshot](#proxmox_snapshot) · [guest_exec](#proxmox_guest_exec) · [vm_manage](#proxmox_vm_manage)

Cluster and platform: [node](#proxmox_node) · [cluster](#proxmox_cluster) · [network](#proxmox_network) · [sdn](#proxmox_sdn) · [storage](#proxmox_storage) · [firewall](#proxmox_firewall) · [access](#proxmox_access) · [acme](#proxmox_acme) · [ceph](#proxmox_ceph) · [ha](#proxmox_ha) · [pool](#proxmox_pool) · [backup](#proxmox_backup)


Shared connection variables
---------------------------

Set once (deployment `group_vars` / `vault.yml`, or `inventory-common` for a
shared PVE cluster group). All roles read them; none redefines them. Each
role's `tasks/main.yml` passes them to the modules through `module_defaults`.

| Name                       | Default | Required                    | Purpose |
|----------------------------|---------|-----------------------------|---------|
| proxmox_api_host           | none    | Yes                         | Any PVE node's FQDN or the cluster VIP; it manages the whole cluster |
| proxmox_api_user           | none    | Yes                         | API user, e.g. `root@pam` |
| proxmox_validate_certs     | none    | Yes                         | Validate the PVE API TLS certificate |
| proxmox_api_password       | omitted | One auth method             | Password for `proxmox_api_user`; use `{{ vault_proxmox_api_password }}` |
| proxmox_api_token_id       | omitted | One auth method             | API token ID, preferred over a password; use a `vault_` variable |
| proxmox_api_token_secret   | omitted | With `proxmox_api_token_id` | API token secret; use a `vault_` variable |
| proxmox_target_vm_name     | `inventory_hostname` | No             | VM that `proxmox_disk`, `proxmox_nic`, `proxmox_vm_manage` and `proxmox_guest_exec` act on; each has its own `<role>_vm_name` defaulting from it |

Notes:
- If both a password and a token are set, the modules prefer the password.
- `proxmox_backup` jobs and `proxmox_guest_exec` use `ansible.builtin.uri` and need **token** auth.
- `proxmox_acme` needs a `root@pam` **password** and has its own variable (`proxmox_acme_api_password`) so setting it doesn't flip every other role to password auth.


proxmox_access
--------------

PVE users, groups, IAM roles, ACLs, authentication realms (LDAP/AD/OpenID) and realm sync.

[`mgcdrd.proxmox.proxmox_access`](roles/proxmox_access/README.md)

**Execution gates**

| Name                           | Default | Required | Purpose |
|--------------------------------|---------|----------|---------|
| proxmox_access_do_user         | `false` | No       | Manage users (tag `user`) |
| proxmox_access_do_group        | `false` | No       | Manage groups (tag `group`) |
| proxmox_access_do_role         | `false` | No       | Manage PVE IAM roles, not Ansible roles (tag `role`) |
| proxmox_access_do_acl          | `false` | No       | Manage ACLs (tag `acl`) |
| proxmox_access_do_domain       | `false` | No       | Manage auth realms (tag `domain`) |
| proxmox_access_do_domain_sync  | `false` | No       | Trigger a realm directory sync (tag `domain_sync`) |

**Section variables**

| Name                                 | Default | Required                                  | Purpose |
|--------------------------------------|---------|-------------------------------------------|---------|
| proxmox_access_users                 | `[]`    | If `proxmox_access_do_user`               | Users; each takes `userid` (with realm, e.g. `jdoe@pve`), `state`, `enable`, `email`, `firstname`, `lastname`, `expire`, `groups`, `password` (PVE-realm, creation only), `tokens` |
| proxmox_access_groups                | `[]`    | If `proxmox_access_do_group`              | Groups; each takes `groupid`, `comment` (creation only), `state`. Membership is set from the user side |
| proxmox_access_roles                 | `[]`    | If `proxmox_access_do_role`               | IAM roles; each takes `roleid`, `privs` (replaced wholesale), `state` |
| proxmox_access_acls                  | `[]`    | If `proxmox_access_do_acl`                | One ACE per item: `path`, `type` (`user`/`group`/`token`), `ugid`, `roleid`, `propagate`, `state` |
| proxmox_access_domains               | `[]`    | If `proxmox_access_do_domain`             | Realms (`ad`/`ldap`/`openid` only); each takes `realm`, `type`, `state`, `default`, `ldap_*`, `ad_*`. `type` is immutable after creation |
| proxmox_access_domain_sync_realm     | `""`    | If `proxmox_access_do_domain_sync`        | Realm to sync |
| proxmox_access_domain_sync_scope     | `""`    | No                                        | `users` \| `groups` \| `both` |
| proxmox_access_domain_sync_enable_new | `""`   | No                                        | Enable newly synced users |
| proxmox_access_domain_sync_remove_vanished | `""` | No                                     | Semicolon list, e.g. `acl;properties;entry`, or `none` |


proxmox_acme
------------

ACME accounts, DNS-01 challenge plugins, and per-node domain/certificate ordering.

[`mgcdrd.proxmox.proxmox_acme`](roles/proxmox_acme/README.md)

**Execution gates**

| Name                          | Default | Required | Purpose |
|-------------------------------|---------|----------|---------|
| proxmox_acme_do_info          | `false` | No       | List accounts and DNS plugins, read-only (tag `info`) |
| proxmox_acme_do_account       | `false` | No       | Manage ACME accounts (tag `account`) |
| proxmox_acme_do_plugin        | `false` | No       | Manage DNS challenge plugins (tag `plugin`) |
| proxmox_acme_do_certificate   | `false` | No       | Set node domains, order/renew/remove certs (tag `certificate`) |

**Section variables**

| Name                      | Default | Required                                      | Purpose |
|---------------------------|---------|-----------------------------------------------|---------|
| proxmox_acme_api_password | `""`    | For account, plugin and certificate sections  | `root@pam` password. The PVE API rejects tokens for these endpoints, so this is independent of `proxmox_api_password` |
| proxmox_acme_accounts     | `[]`    | If `proxmox_acme_do_account`                  | Accounts; each takes `name`, `state`, `contact`, `directory`, `tos`. After creation only `contact` can change |
| proxmox_acme_plugins      | `[]`    | If `proxmox_acme_do_plugin`                   | DNS plugins; each takes `name`, `state`, `plugin` (e.g. `cf`), `data` (provider credentials), `validation_delay` (0-172800 s) |
| proxmox_acme_certificates | `[]`    | If `proxmox_acme_do_certificate`              | Per node: `node_name`, `state`, `account`, `force`, `domains` (list of `domain` + optional `plugin`; omit plugin for HTTP-01) |

Facts set by the info section: `proxmox_acme_accounts_facts`, `proxmox_acme_plugins_facts`.


proxmox_backup
--------------

Scheduled backup jobs, per-VM job membership, and on-demand backups.

[`mgcdrd.proxmox.proxmox_backup`](roles/proxmox_backup/README.md)

**Execution gates**

| Name                         | Default | Required | Purpose |
|------------------------------|---------|----------|---------|
| proxmox_backup_do_info       | `false` | No       | Query jobs or per-VM membership (tag `info`) |
| proxmox_backup_do_job        | `false` | No       | Create/update/delete scheduled jobs. Needs API token auth (tag `job`) |
| proxmox_backup_do_membership | `false` | No       | Toggle a VM's membership in an existing job (tag `membership`) |
| proxmox_backup_do_run        | `false` | No       | Trigger a backup now. Not idempotent (tag `run`) |

**info**

| Name                      | Default | Required | Purpose |
|---------------------------|---------|----------|---------|
| proxmox_backup_info_vm_name | `""`  | No       | VM to query; mutually exclusive with `vm_id` and `jobs` |
| proxmox_backup_info_vm_id   | `""`  | No       | VMID to query; mutually exclusive with `vm_name` and `jobs` |
| proxmox_backup_info_jobs    | `false` | No     | `true` returns the raw job list; `false` returns the per-VM view |

**job / membership**

| Name                      | Default | Required                        | Purpose |
|---------------------------|---------|---------------------------------|---------|
| proxmox_backup_jobs       | `[]`    | If `proxmox_backup_do_job`      | Jobs; each needs an `id` (idempotency key). Also `state`, `comment`, `schedule`, `storage`, `vmid` (comma-separated string), `all`, `exclude`, `pool`, `node`, `mode`, `compress`, `prune_backups`, `enabled`, `repeat_missed`, `notes_template`, `bwlimit`, `notification_mode` |
| proxmox_backup_membership | `[]`    | If `proxmox_backup_do_membership` | Items of `vm_name` or `vm_id`, `backup_id` (always set it), `state` |

**run**

| Name                            | Default      | Required                 | Purpose |
|---------------------------------|--------------|--------------------------|---------|
| proxmox_backup_run_mode         | `include`    | No                       | `include` \| `all` \| `pool` |
| proxmox_backup_run_storage      | `""`         | No                       | Target storage |
| proxmox_backup_run_vmids        | `[]`         | If mode is `include`     | VMIDs to back up |
| proxmox_backup_run_pool         | `""`         | If mode is `pool`        | Pool to back up |
| proxmox_backup_run_backup_mode  | `snapshot`   | No                       | `snapshot` \| `suspend` \| `stop` |
| proxmox_backup_run_compress     | `""`         | No                       | Compression for this run |
| proxmox_backup_run_description  | `""`         | No                       | Backup notes |
| proxmox_backup_run_retention    | `""`         | No                       | Overrides the cluster retention for this run only |
| proxmox_backup_run_wait         | `false`      | No                       | Poll until the backup task completes |
| proxmox_backup_run_wait_timeout | `10`         | No                       | Wait timeout when `proxmox_backup_run_wait` is `true` |


proxmox_ceph
------------

Ceph monitors, managers, metadata servers, OSDs and pools through the PVE API.

[`mgcdrd.proxmox.proxmox_ceph`](roles/proxmox_ceph/README.md)

**Execution gates**

| Name                 | Default | Required | Purpose |
|----------------------|---------|----------|---------|
| proxmox_ceph_do_mon  | `false` | No       | Monitors (tag `mon`) |
| proxmox_ceph_do_mgr  | `false` | No       | Managers (tag `mgr`) |
| proxmox_ceph_do_mds  | `false` | No       | Metadata servers (tag `mds`) |
| proxmox_ceph_do_osd  | `false` | No       | OSDs (tag `osd`) |
| proxmox_ceph_do_pool | `false` | No       | Pools (tag `pool`) |

**Section variables**

| Name               | Default | Required                  | Purpose |
|--------------------|---------|---------------------------|---------|
| proxmox_ceph_mons  | `[]`    | If `proxmox_ceph_do_mon`  | Items of `node`, `state` (one monitor per node) |
| proxmox_ceph_mgrs  | `[]`    | If `proxmox_ceph_do_mgr`  | Items of `node`, `state` |
| proxmox_ceph_mds   | `[]`    | If `proxmox_ceph_do_mds`  | Items of `node`, `state` (CephFS) |
| proxmox_ceph_osds  | `[]`    | If `proxmox_ceph_do_osd`  | Items of `node`, `state` (`present` \| `absent` \| `in` \| `out` \| `scrub` \| `start` \| `stop` \| `restart`). `dev` required for `present`, `osdid` for every other state. Also `crush_device_class`, `db_dev`, `db_dev_size`, `wal_dev`, `wal_dev_size`, `encrypted`, `osds_per_device`, `cleanup` |
| proxmox_ceph_pools | `[]`    | If `proxmox_ceph_do_pool` | Items of `node`, `name`, `state`, `size`, `min_size`, `pg_num`, `pg_autoscale_mode` (`on` \| `off` \| `warn`), `crush_rule`, `add_storages`, `timeout` |


proxmox_cluster
---------------

Cluster status, cluster creation, and node join over the PVE API (no `pvecm`).

[`mgcdrd.proxmox.proxmox_cluster`](roles/proxmox_cluster/README.md)

**Execution gates**

| Name                      | Default | Required | Purpose |
|---------------------------|---------|----------|---------|
| proxmox_cluster_do_info   | `false` | No       | Cluster status and join info, read-only (tag `info`) |
| proxmox_cluster_do_create | `false` | No       | Create a new cluster (tag `create`) |
| proxmox_cluster_do_join   | `false` | No       | Join nodes to an existing cluster (tag `join`) |

**Section variables**

| Name                                 | Default | Required                       | Purpose |
|--------------------------------------|---------|--------------------------------|---------|
| proxmox_cluster_name                 | `""`    | If `proxmox_cluster_do_create` | Cluster name; `proxmox_api_host` must be the intended master |
| proxmox_cluster_link0                | `""`    | No                             | Ring 0 address |
| proxmox_cluster_link1                | `""`    | No                             | Ring 1 address |
| proxmox_cluster_master_ip            | `""`    | If `proxmox_cluster_do_join`   | Existing master, reachable from every joining node |
| proxmox_cluster_master_fingerprint   | `""`    | If `proxmox_cluster_do_join`   | Master's fingerprint, from `proxmox_cluster_join_facts.cluster_join[0].nodelist[].pve_fp` |
| proxmox_cluster_master_api_password  | `{{ vault_proxmox_cluster_master_api_password \| default(omit) }}` | If `proxmox_cluster_do_join` | Master's password for the join call. Set it explicitly; the module's fallback reuses the local node's credential |
| proxmox_cluster_join_nodes           | `[]`    | If `proxmox_cluster_do_join`   | Nodes to add; each needs `api_host` (the new node's own API) and takes `link0`, `link1`, and optional per-node `api_user` / `api_password` / `api_token_id` / `api_token_secret` |

Facts set by the info section: `proxmox_cluster_status`, `proxmox_cluster_join_facts`.


proxmox_disk
------------

Adds or resizes a VM disk in PVE and reports the new guest block device.

[`mgcdrd.proxmox.proxmox_disk`](roles/proxmox_disk/README.md)

| Name                       | Default                                                  | Required | Purpose |
|----------------------------|----------------------------------------------------------|----------|---------|
| proxmox_disk_vm_name       | `{{ proxmox_target_vm_name \| default(inventory_hostname) }}` | No  | VM to act on; VMID and node are discovered by name |
| proxmox_disk_slot          | `virtio1`                                                | No       | PVE disk slot. Must not already be occupied for `present` |
| proxmox_disk_storage       | `""`                                                     | Yes      | PVE storage backend, e.g. `local-lvm` |
| proxmox_disk_size          | `"400"`                                                  | No       | GiB integer for `present`; `+XG` form for `resized` |
| proxmox_disk_state         | `present`                                                | No       | `present` adds a disk \| `resized` grows an existing one |
| proxmox_disk_boot_devices  | `[sda]`                                                  | No       | Boot devices excluded when finding the data disk on an idempotent re-run |

Result: the role sets the host fact `proxmox_disk_device` (e.g. `/dev/vdb`) for downstream roles such as `lvm2_provision`.


proxmox_firewall
----------------

Cluster and node firewall options, rules, security groups, aliases and IP sets.

[`mgcdrd.proxmox.proxmox_firewall`](roles/proxmox_firewall/README.md)

**Execution gates**

| Name                                | Default | Required | Purpose |
|-------------------------------------|---------|----------|---------|
| proxmox_firewall_do_info            | `false` | No       | Read rules, groups, aliases, ipsets (tag `info`) |
| proxmox_firewall_do_cluster_options | `false` | No       | Cluster-wide enable/policy/log rate (tag `cluster_options`) |
| proxmox_firewall_do_node_options    | `false` | No       | Per-node options (tag `node_options`) |
| proxmox_firewall_do_rule            | `false` | No       | Firewall rules (tag `rule`) |
| proxmox_firewall_do_group           | `false` | No       | Security groups, name only (tag `group`) |
| proxmox_firewall_do_alias           | `false` | No       | Aliases (tag `alias`) |
| proxmox_firewall_do_ipset           | `false` | No       | IP sets (tag `ipset`) |

**info**

| Name                                       | Default     | Required                   | Purpose |
|--------------------------------------------|-------------|----------------------------|---------|
| proxmox_firewall_info_level                | `cluster`   | No                         | `cluster` \| `group` \| `vnet` \| `node` \| `vm` |
| proxmox_firewall_info_node                 | `""`        | If level is `node`         | Node name |
| proxmox_firewall_info_vmid                 | `""`        | If level is `vm`           | VMID |
| proxmox_firewall_info_vnet                 | `""`        | If level is `vnet`         | VNet name |
| proxmox_firewall_info_group                | `""`        | If level is `group`        | Security group name |
| proxmox_firewall_info_node_options_target  | `""`        | No                         | Node whose options to read; `""` skips the lookup |

**cluster_options**

| Name                                     | Default    | Required | Purpose |
|------------------------------------------|------------|----------|---------|
| proxmox_firewall_cluster_state           | `disabled` | No       | `enabled` \| `disabled` |
| proxmox_firewall_cluster_ebtables        | `true`     | No       | Enable ebtables |
| proxmox_firewall_cluster_input_policy    | `DROP`     | No       | `ACCEPT` \| `REJECT` \| `DROP` |
| proxmox_firewall_cluster_output_policy   | `ACCEPT`   | No       | `ACCEPT` \| `REJECT` \| `DROP` |
| proxmox_firewall_cluster_forward_policy  | `ACCEPT`   | No       | `ACCEPT` \| `DROP` |
| proxmox_firewall_cluster_log_ratelimit   | `{}`       | No       | Dict of `enabled`, `burst`, `rate` |

**Section variables**

| Name                          | Default | Required                                 | Purpose |
|-------------------------------|---------|------------------------------------------|---------|
| proxmox_firewall_node_options | `[]`    | If `proxmox_firewall_do_node_options`    | Per node: `node_name`, `state`, `log_level_in`, `nosmurfs`, `protection_synflood`, and related options |
| proxmox_firewall_rules        | `[]`    | If `proxmox_firewall_do_rule`            | Per level (`cluster`/`group`/`vnet`/`node`/`vm`, plus `vmid` etc.): `state` and a `rules` list of `type`, `action`, `pos` (required), `proto`, `dport`, `log`, `enable` |
| proxmox_firewall_groups       | `[]`    | If `proxmox_firewall_do_group`           | `group`, `comment`, `state`. Name only; rule content needs `proxmox_cluster_firewall_security_group`, not wired in yet |
| proxmox_firewall_aliases      | `[]`    | If `proxmox_firewall_do_alias`           | `name`, `cidr`, `state` |
| proxmox_firewall_ipsets       | `[]`    | If `proxmox_firewall_do_ipset`           | `name`, `comment`, `state`, `cidrs` (list of `cidr` + `comment`) |

Note: `cluster_options`, `node_options` and the node part of `info` need `community.proxmox` >= 2.1.0 (the `*_firewall_options` module names).


proxmox_guest_exec
------------------

Runs a command inside a VM through the QEMU guest agent (raw API, token auth only).

[`mgcdrd.proxmox.proxmox_guest_exec`](roles/proxmox_guest_exec/README.md)

| Name                               | Default                                                  | Required | Purpose |
|------------------------------------|----------------------------------------------------------|----------|---------|
| proxmox_guest_exec_vm_name         | `{{ proxmox_target_vm_name \| default(inventory_hostname) }}` | No  | VM to run in; looked up by name unless both `vmid` and `node` are set |
| proxmox_guest_exec_vmid            | `""`                                                     | No       | Set with `node` to skip the lookup |
| proxmox_guest_exec_node            | `""`                                                     | No       | Set with `vmid` to skip the lookup |
| proxmox_guest_exec_command         | `[]`                                                     | Yes      | argv list, first element is the executable. No shell involved |
| proxmox_guest_exec_input_data      | `""`                                                     | No       | Data piped to the command's stdin |
| proxmox_guest_exec_fail_on_nonzero | `true`                                                   | No       | Fail the play on a non-zero exit |
| proxmox_guest_exec_timeout         | `60`                                                     | No       | Seconds to wait for completion |
| proxmox_guest_exec_poll_interval   | `2`                                                      | No       | Seconds between polls |
| proxmox_guest_exec_no_log          | `true`                                                   | No       | Hide request/response on the submit and poll tasks. Set `false` only to debug |

Result: `proxmox_guest_exec_result`. Requires `proxmox_api_token_id` and `proxmox_api_token_secret`.


proxmox_ha
----------

HA groups (legacy), HA resources, and PVE 9+ node/resource-affinity rules.

[`mgcdrd.proxmox.proxmox_ha`](roles/proxmox_ha/README.md)

**Execution gates**

| Name                    | Default | Required | Purpose |
|-------------------------|---------|----------|---------|
| proxmox_ha_do_info      | `false` | No       | Query HA rules only (tag `info`) |
| proxmox_ha_do_group     | `false` | No       | Legacy HA groups, deprecated on PVE 9+ (tag `group`) |
| proxmox_ha_do_resource  | `false` | No       | HA resources (tag `resource`) |
| proxmox_ha_do_rule      | `false` | No       | PVE 9+ HA rules (tag `rule`) |

**Section variables**

| Name                     | Default | Required                         | Purpose |
|--------------------------|---------|----------------------------------|---------|
| proxmox_ha_groups        | `[]`    | If `proxmox_ha_do_group`         | `name`, `state`, `nodes` (list or `node:prio,...`), `nofailback`, `restricted`, `comment` |
| proxmox_ha_resources     | `[]`    | If `proxmox_ha_do_resource`      | `name` (the sid, e.g. `vm:100`), `state`, `group`, `max_relocate`, `max_restart`, `hastate` (`started` \| `stopped` \| `disabled` \| `ignored`), `comment` |
| proxmox_ha_rules         | `[]`    | If `proxmox_ha_do_rule`          | `name`, `state`, `type` (`node-affinity` \| `resource-affinity`), `nodes`, `resources`, `strict`, `affinity` (`positive` \| `negative`), `disable`, `comment` |
| proxmox_ha_info_rule     | `""`    | No                               | Filter to one rule; mutually exclusive with `type` / `resource` |
| proxmox_ha_info_type     | `""`    | No                               | `node-affinity` \| `resource-affinity` |
| proxmox_ha_info_resource | `""`    | No                               | Rules affecting a resource id, e.g. `vm:100` |


proxmox_lxc
-----------

LXC containers: create from a template, clone, lifecycle, and convert to a template.

[`mgcdrd.proxmox.proxmox_lxc`](roles/proxmox_lxc/README.md)

**Execution gates**

| Name                     | Default | Required | Purpose |
|--------------------------|---------|----------|---------|
| proxmox_lxc_do_info      | `false` | No       | Cluster/node/storage facts, read-only (tag `info`) |
| proxmox_lxc_do_create    | `false` | No       | Create from a vztmpl (tag `create`) |
| proxmox_lxc_do_clone     | `false` | No       | Clone an existing container (tag `clone`) |
| proxmox_lxc_do_lifecycle | `false` | No       | Start/stop/restart/destroy (tag `lifecycle`) |
| proxmox_lxc_do_template  | `false` | No       | Convert to a template (tag `template`) |

**create**

| Name                            | Default                                        | Required | Purpose |
|---------------------------------|------------------------------------------------|----------|---------|
| proxmox_lxc_vmid                | `""`                                           | No       | `""` lets PVE assign the next free VMID |
| proxmox_lxc_node                | `""`                                           | Yes      | Node to create on |
| proxmox_lxc_hostname            | `""`                                           | Yes      | Container hostname |
| proxmox_lxc_ostemplate          | `""`                                           | Yes      | Template, e.g. `local:vztmpl/rockylinux-9-default_20240523_amd64.tar.xz` |
| proxmox_lxc_password            | `{{ vault_proxmox_lxc_password \| default(omit) }}` | No  | Root password |
| proxmox_lxc_pool                | `""`                                           | No       | Resource pool |
| proxmox_lxc_storage             | `local-lvm`                                    | No       | Root disk storage |
| proxmox_lxc_disk_size           | `"8"`                                          | No       | GiB, no suffix |
| proxmox_lxc_cores               | `2`                                            | No       | CPU cores |
| proxmox_lxc_memory              | `1024`                                         | No       | MiB |
| proxmox_lxc_swap                | `512`                                          | No       | MiB |
| proxmox_lxc_net0                | `name=eth0,bridge=vmbr0,ip=dhcp`               | No       | First NIC definition |
| proxmox_lxc_nameserver          | `""`                                           | No       | DNS server |
| proxmox_lxc_searchdomain        | `""`                                           | No       | DNS search domain |
| proxmox_lxc_pubkey              | `""`                                           | No       | Public key content, not a path |
| proxmox_lxc_unprivileged        | `true`                                         | No       | Unprivileged container |
| proxmox_lxc_features            | `[]`                                           | No       | e.g. `["nesting=1", "keyctl=1"]` |
| proxmox_lxc_onboot              | `true`                                         | No       | Start at boot |
| proxmox_lxc_tags                | `[]`                                           | No       | PVE tags |
| proxmox_lxc_create_timeout      | `300`                                          | No       | Seconds |
| proxmox_lxc_start_after_create  | `true`                                         | No       | Start after creation |

**clone**

| Name                          | Default         | Required | Purpose |
|-------------------------------|-----------------|----------|---------|
| proxmox_lxc_clone_source_name | `""`            | Yes      | Container to clone |
| proxmox_lxc_clone_node        | `""`            | No       | `""` = the source's node |
| proxmox_lxc_clone_vmid        | `""`            | No       | `""` lets PVE pick |
| proxmox_lxc_clone_hostname    | `""`            | Yes      | Hostname of the new container |
| proxmox_lxc_clone_type        | `opportunistic` | No       | `full` \| `linked` \| `opportunistic`; `linked` needs a template source |
| proxmox_lxc_clone_storage     | `""`            | For a full clone | Target storage |
| proxmox_lxc_clone_timeout     | `300`           | No       | Seconds |
| proxmox_lxc_clone_start_after | `true`          | No       | Start after cloning |

**lifecycle / template**

| Name                                    | Default | Required | Purpose |
|-----------------------------------------|---------|----------|---------|
| proxmox_lxc_target_name                 | `""`    | Yes      | Container name, discovered via `proxmox_vm_info` |
| proxmox_lxc_lifecycle_state             | `""`    | Yes      | `started` \| `stopped` \| `restarted` \| `absent` |
| proxmox_lxc_force_stop                  | `false` | No       | `stopped` only: power off instead of graceful shutdown |
| proxmox_lxc_purge                       | `false` | No       | `absent` only: also remove from backup, replication and HA config |
| proxmox_lxc_destroy_unreferenced_disks  | `true`  | No       | `absent` only |
| proxmox_lxc_lifecycle_timeout           | `300`   | No       | Seconds |
| proxmox_lxc_template_source_name        | `""`    | If `proxmox_lxc_do_template` | Container to convert |
| proxmox_lxc_template_force              | `true`  | No       | Stop the container first if it is running |

Facts set by the info section: `proxmox_lxc_cluster_status`, `proxmox_lxc_node_facts`, `proxmox_lxc_storage_facts`.


proxmox_network
---------------

Node network interfaces (bridges, bonds, VLANs) and applying staged changes.

[`mgcdrd.proxmox.proxmox_network`](roles/proxmox_network/README.md)

**Execution gates**

| Name                     | Default | Required | Purpose |
|--------------------------|---------|----------|---------|
| proxmox_network_do_info  | `false` | No       | Read interfaces or pending changes (tag `info`) |
| proxmox_network_do_iface | `false` | No       | Create/modify/delete interfaces (tag `iface`) |
| proxmox_network_do_apply | `false` | No       | Apply staged changes live (tag `apply`) |

**Section variables**

| Name                                | Default | Required                          | Purpose |
|-------------------------------------|---------|-----------------------------------|---------|
| proxmox_network_info_node           | `""`    | If `proxmox_network_do_info`      | Node to read |
| proxmox_network_info_iface          | `""`    | No                                | `""` = all interfaces |
| proxmox_network_info_iface_type     | `""`    | No                                | `""` = all types |
| proxmox_network_info_check_changes  | `false` | No                                | `true` reports pending (unapplied) changes instead of live config |
| proxmox_network_node                | `""`    | If `proxmox_network_do_iface`     | Default `node` for every interface item; override per item with `node` |
| proxmox_network_interfaces          | `[]`    | If `proxmox_network_do_iface`     | Each item needs `iface`; the rest follows `community.proxmox.proxmox_node_network` (`iface_type`, `state`, `cidr`, `gateway`, `bridge_ports`, `bridge_vlan_aware`, `bond_mode`, `slaves`, `autostart`, ...) |
| proxmox_network_apply_node          | `""`    | If `proxmox_network_do_apply`     | Node whose staged changes are applied |


proxmox_nic
-----------

Adds, changes or removes VM network interfaces (`net0`-`net31`).

[`mgcdrd.proxmox.proxmox_nic`](roles/proxmox_nic/README.md)

| Name                  | Default                                                  | Required | Purpose |
|-----------------------|----------------------------------------------------------|----------|---------|
| proxmox_nic_vm_name   | `{{ proxmox_target_vm_name \| default(inventory_hostname) }}` | No  | VM to act on; VMID discovered by name |
| proxmox_nic_vmid      | `""`                                                     | No       | Set to skip discovery, e.g. right after a `proxmox_vm` clone |
| proxmox_nic_interfaces | `[]`                                                    | Yes      | One item per slot: `interface` (`net1`...), `bridge`, `model` (default `virtio`), `tag` (VLAN), `firewall`, `mtu`, `mac`, `state` (`present` \| `absent`). Deleting needs only `interface` + `state: absent` |

`net0` is normally set at clone time by `proxmox_vm_net0`; list it here only for post-creation changes.


proxmox_node
------------

Node status, power state, certificates/DNS/subscription, and task listing.

[`mgcdrd.proxmox.proxmox_node`](roles/proxmox_node/README.md)

**Execution gates**

| Name                    | Default | Required | Purpose |
|-------------------------|---------|----------|---------|
| proxmox_node_do_info    | `false` | No       | List nodes with status, read-only (tag `info`) |
| proxmox_node_do_power   | `false` | No       | Wake-on-LAN or shutdown (tag `power`) |
| proxmox_node_do_config  | `false` | No       | Certificates, DNS, subscription (tag `config`) |
| proxmox_node_do_task    | `false` | No       | List/poll PVE tasks, read-only (tag `task`) |

**Section variables**

| Name                          | Default   | Required                         | Purpose |
|-------------------------------|-----------|----------------------------------|---------|
| proxmox_node_target           | `""`      | If `proxmox_node_do_power`       | Node name |
| proxmox_node_power_state      | `""`      | If `proxmox_node_do_power`       | `online` (WOL if not already up) \| `offline` (graceful shutdown). There is no reboot |
| proxmox_node_config_target    | `""`      | If `proxmox_node_do_config`      | Node name |
| proxmox_node_certificates     | `{}`      | No                               | `certificate_file_path` or `certificate_raw`, `private_key_file_path` or `private_key_raw`, `state`, `force`, `restart` |
| proxmox_node_dns              | `{}`      | No                               | `dns1`, `dns2`, `search` (`search` required when used) |
| proxmox_node_subscription     | `{}`      | No                               | `state`, `key` (use a `vault_` variable) |
| proxmox_node_task_target      | `""`      | If `proxmox_node_do_task`        | Node name |
| proxmox_node_task_upid        | `""`      | No                               | `""` lists tasks; set to poll one |
| proxmox_node_task_source      | `archive` | No                               | `archive` (finished) \| `active` (running) \| `all` |
| proxmox_node_task_wait        | `false`   | No                               | Poll `proxmox_node_task_upid` until it has a status |
| proxmox_node_task_wait_retries | `30`     | No                               | Poll attempts |
| proxmox_node_task_wait_delay  | `5`       | No                               | Seconds between polls |

Fact set by the info section: `proxmox_node_facts`.


proxmox_pool
------------

Resource pools and their VM/storage membership.

[`mgcdrd.proxmox.proxmox_pool`](roles/proxmox_pool/README.md)

**Execution gates**

| Name                    | Default | Required | Purpose |
|-------------------------|---------|----------|---------|
| proxmox_pool_do_pool    | `false` | No       | Create/delete pools (tag `pool`) |
| proxmox_pool_do_member  | `false` | No       | Manage membership (tag `member`) |

**Section variables**

| Name                      | Default   | Required                      | Purpose |
|---------------------------|-----------|-------------------------------|---------|
| proxmox_pool_pools        | `[]`      | If `proxmox_pool_do_pool`     | `poolid`, `comment` (creation only), `state`. A pool must be empty before deletion |
| proxmox_pool_poolid       | `""`      | If `proxmox_pool_do_member`   | Target pool |
| proxmox_pool_members      | `[]`      | If `proxmox_pool_do_member`   | Items of `{vm: <vmid-or-name>}` or `{storage: <name>}` |
| proxmox_pool_member_state | `present` | No                            | `present` \| `absent`. Ignored when `proxmox_pool_exclusive` is `true` |
| proxmox_pool_exclusive    | `false`   | No                            | `true` reconciles membership to exactly match `proxmox_pool_members` |
| proxmox_pool_allow_move   | `false`   | No                            | Allow adding a guest that is already in another pool |


proxmox_sdn
-----------

SDN zones, VNets and subnets. Changes apply immediately; there is no separate apply step.

[`mgcdrd.proxmox.proxmox_sdn`](roles/proxmox_sdn/README.md)

**Execution gates**

| Name                  | Default | Required | Purpose |
|-----------------------|---------|----------|---------|
| proxmox_sdn_do_info   | `false` | No       | List zones and VNets, read-only (tag `info`) |
| proxmox_sdn_do_zone   | `false` | No       | Zones (tag `zone`) |
| proxmox_sdn_do_vnet   | `false` | No       | VNets (tag `vnet`) |
| proxmox_sdn_do_subnet | `false` | No       | Subnets (tag `subnet`) |

**Section variables**

| Name               | Default | Required                    | Purpose |
|--------------------|---------|-----------------------------|---------|
| proxmox_sdn_zones  | `[]`    | If `proxmox_sdn_do_zone`    | `zone` and `type` required; also `bridge`, `fabric`, `state`. `type` is immutable once created |
| proxmox_sdn_vnets  | `[]`    | If `proxmox_sdn_do_vnet`    | `vnet` and `zone` required; also `tag`, `vlanaware`, `state` |
| proxmox_sdn_subnets | `[]`   | If `proxmox_sdn_do_subnet`  | `subnet`, `vnet` and `zone` required (even for `absent`); also `gateway`, `snat`, `dhcp_range` (list of `start` / `end`), `state` |

Facts set by the info section: `proxmox_sdn_zone_facts`, `proxmox_sdn_vnet_facts`.


proxmox_snapshot
----------------

Create, delete and roll back snapshots for VMs and containers.

[`mgcdrd.proxmox.proxmox_snapshot`](roles/proxmox_snapshot/README.md)

**Execution gates**

| Name                        | Default | Required | Purpose |
|-----------------------------|---------|----------|---------|
| proxmox_snapshot_do_info    | `false` | No       | List/query snapshots, read-only (tag `info`) |
| proxmox_snapshot_do_manage  | `false` | No       | Create/delete/rollback (tag `manage`) |

**Section variables**

| Name                          | Default | Required                          | Purpose |
|-------------------------------|---------|-----------------------------------|---------|
| proxmox_snapshot_snapshots    | `[]`    | If `proxmox_snapshot_do_manage`   | Each item sets `hostname` or `vmid` (`vmid` wins) and `snapname` (always set it). Also `state` (`present` \| `absent` \| `rollback`), `description`, `vmstate` (QEMU only), `retention` (keep N newest; `0` = all), `force` (`absent` only), `unbind` (LXC only, needs `root@pam` password), `timeout` |
| proxmox_snapshot_info_hostname | `""`   | One of hostname / vmid            | Target by name |
| proxmox_snapshot_info_vmid    | `""`    | One of hostname / vmid            | Target by VMID |
| proxmox_snapshot_info_snapname | `""`   | No                                | `""` lists all snapshots for the target |


proxmox_storage
---------------

Storage backend definitions (create/delete only; no update path).

[`mgcdrd.proxmox.proxmox_storage`](roles/proxmox_storage/README.md)

**Execution gates**

| Name                       | Default | Required | Purpose |
|----------------------------|---------|----------|---------|
| proxmox_storage_do_info    | `false` | No       | List configured storage, read-only (tag `info`) |
| proxmox_storage_do_manage  | `false` | No       | Create/delete storage definitions (tag `manage`) |

**Section variables**

| Name                         | Default | Required                         | Purpose |
|------------------------------|---------|----------------------------------|---------|
| proxmox_storage_definitions  | `[]`    | If `proxmox_storage_do_manage`   | Each item takes `name`, `type`, `state`, `nodes` (omit for cluster-wide), `content`, plus one options dict matching `type` (below). `present` on an existing storage is always a no-op; delete and recreate to change it |

**`type` and its options dict**

| type      | Options dict       | Required keys |
|-----------|--------------------|---------------|
| cephfs    | `cephfs_options`   | `path`, `monhost`; also `subdir`, `username`, `password`, `client_keyring`, `fs_name` |
| cifs      | `cifs_options`     | `server`, `share`, `username`, `password`; also `domain`, `subdir`, `smb_version` |
| dir       | `dir_options`      | `path` |
| iscsi     | `iscsi_options`    | `portal`, `target` |
| nfs       | `nfs_options`      | `server`, `export`; also `options`, `preallocation` |
| pbs       | `pbs_options`      | `server`, `datastore`, `username`, `password`; also `namespace`, `fingerprint`, `encryption_key` |
| rbd       | `rbd_options`      | `pool` (enforced by the PVE API) |
| zfspool   | `zfspool_options`  | `pool`; also `sparse` |

No `lvm`, `lvmthin` or `btrfs` type is available in this module.

Fact set by the info section: `proxmox_storage_facts`.


proxmox_vm
----------

QEMU VMs: clone from a template, lifecycle, and convert to a template.

[`mgcdrd.proxmox.proxmox_vm`](roles/proxmox_vm/README.md)

**Execution gates**

| Name                     | Default | Required | Purpose |
|--------------------------|---------|----------|---------|
| proxmox_vm_do_info       | `false` | No       | Cluster/node/storage facts, read-only (tag `info`) |
| proxmox_vm_do_clone      | `false` | No       | Clone a VM (tag `clone`) |
| proxmox_vm_do_lifecycle  | `false` | No       | Start/stop/destroy (tag `lifecycle`) |
| proxmox_vm_do_template   | `false` | No       | Convert to a template (tag `template`) |

**clone**

| Name                           | Default                | Required | Purpose |
|--------------------------------|------------------------|----------|---------|
| proxmox_vm_clone_source        | `""`                   | Yes      | Template or VM to clone |
| proxmox_vm_name                | `""`                   | Yes      | Name of the new VM |
| proxmox_vm_node                | `""`                   | Yes      | Node to place the clone on |
| proxmox_vm_newid               | `""`                   | No       | Explicit VMID; `""` lets PVE pick |
| proxmox_vm_full_clone          | `true`                 | No       | `false` = linked clone (template source only) |
| proxmox_vm_storage             | `""`                   | Yes      | Target storage |
| proxmox_vm_format              | `qcow2`                | No       | `qcow2` \| `raw` \| `vmdk`, full clone only |
| proxmox_vm_clone_timeout       | `600`                  | No       | Seconds |
| proxmox_vm_start_after_clone   | `true`                 | No       | Start after cloning |

**Config applied after the clone**

| Name                        | Default                                          | Required | Purpose |
|-----------------------------|--------------------------------------------------|----------|---------|
| proxmox_vm_cores            | `2`                                              | No       | CPU cores |
| proxmox_vm_sockets          | `1`                                              | No       | CPU sockets |
| proxmox_vm_memory           | `2048`                                           | No       | MiB |
| proxmox_vm_net0             | `virtio,bridge=vmbr0`                            | No       | First NIC definition |
| proxmox_vm_agent            | `true`                                           | No       | QEMU guest agent |
| proxmox_vm_onboot           | `true`                                           | No       | Start at boot |
| proxmox_vm_tags             | `[]`                                             | No       | PVE tags |
| proxmox_vm_update_unsafe    | `true`                                           | No       | Allow net/disk-bus updates on the follow-up call (needed for `net0` to apply); `false` leaves `net0` as the template has it |

**Cloud-init** (leave blank to skip)

| Name                        | Default                                          | Required | Purpose |
|-----------------------------|--------------------------------------------------|----------|---------|
| proxmox_vm_ipconfig0        | `""`                                             | No       | e.g. `ip=dhcp` or `ip=10.10.10.50/24,gw=10.10.10.1` |
| proxmox_vm_ciuser           | `""`                                             | No       | Cloud-init user |
| proxmox_vm_cipassword       | `{{ vault_proxmox_vm_cipassword \| default(omit) }}` | No | Cloud-init password |
| proxmox_vm_sshkeys          | `""`                                             | No       | Public key content, not a path |
| proxmox_vm_nameservers      | `[]`                                             | No       | DNS servers |
| proxmox_vm_searchdomains    | `[]`                                             | No       | DNS search domains |

**lifecycle / template**

| Name                                  | Default | Required                  | Purpose |
|---------------------------------------|---------|---------------------------|---------|
| proxmox_vm_target_name                | `""`    | If `proxmox_vm_do_lifecycle` | VM name, discovered via `proxmox_vm_info` |
| proxmox_vm_lifecycle_state            | `""`    | If `proxmox_vm_do_lifecycle` | `started` \| `stopped` \| `absent` |
| proxmox_vm_force_stop                 | `false` | No                        | `stopped` only: power off instead of graceful shutdown |
| proxmox_vm_destroy_unreferenced_disks | `true`  | No                        | `absent` only |
| proxmox_vm_lifecycle_timeout          | `300`   | No                        | Seconds |
| proxmox_vm_template_source_name       | `""`    | If `proxmox_vm_do_template`  | Stopped VM to convert |

Facts set by the info section: `proxmox_vm_cluster_status`, `proxmox_vm_node_facts`, `proxmox_vm_storage_facts`.


proxmox_vm_manage
-----------------

Wrapper that applies disks, NICs and snapshots to one VM by calling `proxmox_disk`, `proxmox_nic` and `proxmox_snapshot`.

[`mgcdrd.proxmox.proxmox_vm_manage`](roles/proxmox_vm_manage/README.md)

| Name                          | Default                                                  | Required | Purpose |
|-------------------------------|----------------------------------------------------------|----------|---------|
| proxmox_vm_manage_vm_name     | `{{ proxmox_target_vm_name \| default(inventory_hostname) }}` | No  | VM to manage |
| proxmox_vm_manage_vmid        | `""`                                                     | No       | Skips the by-name lookup for the NIC and snapshot sub-roles; the disk sub-role always looks up by name |
| proxmox_vm_manage_disks       | `[]`                                                     | No       | Items of `slot`, `storage`, `size`, `state` (`present` \| `resized`). Each item runs a full `proxmox_disk` call, so N items means N lookups. PVE side only; grow the guest filesystem with `lvm2` afterwards |
| proxmox_vm_manage_nics        | `[]`                                                     | No       | Passed straight to `proxmox_nic_interfaces` |
| proxmox_vm_manage_snapshots   | `[]`                                                     | No       | Passed straight to `proxmox_snapshot_snapshots` |
