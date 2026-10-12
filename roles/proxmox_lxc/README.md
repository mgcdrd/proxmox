proxmox_lxc
===========

Manages LXC container lifecycle in ProxMox VE via its REST API: create a
container from a template, clone one from an existing container, start/stop/
restart/destroy an existing container, convert a container into a template,
and query cluster/node/storage info. Parallels `mgcdrd.proxmox.proxmox_vm`
but for containers instead of QEMU VMs.

Tested on: any Ansible controller against a ProxMox VE 8.x cluster.


Requirements
------------

- The Ansible controller must be able to reach the PVE API host on port 8006.
- `community.proxmox` collection installed (`>= 1.4.0` — the SDN/network
  modules used elsewhere in this collection landed there; the LXC module
  itself has been present since the collection's first release).
- **`proxmoxer >= 2.3` and `requests` must be installed on the Ansible
  controller**:
  - RPM: `pip3 install 'proxmoxer>=2.3' requests`
  - Debian 12+ (system Ansible): `pip3 install 'proxmoxer>=2.3' requests --break-system-packages`
  - Debian 12+ (Ansible in a venv): same command inside the venv


Authentication
---------------

Two methods are supported — set one pair of variables, leave the other unset.
These are the same `proxmox_api_token_id`/`_secret`/`_user`/`_password`
vars every `proxmox_*` role uses — set them once, not per role:

```yaml
# API token (recommended)
proxmox_api_token_id:     "{{ vault_proxmox_api_token_id }}"
proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"

# Password
proxmox_api_user:     "root@pam"
proxmox_api_password: "{{ vault_proxmox_api_password }}"
```

Unset `vault_*` variables resolve to `omit`, so only the pair you configure
is ever sent to the module. Grant the token at minimum `VM.Allocate`,
`VM.Clone`, `VM.Config.*`, `VM.PowerMgmt`, `Datastore.AllocateSpace` on the
relevant path.


Role Variables
---------------

### Variable reference

| Variable | Default | Required | Description |
|---|---|---|---|
| `proxmox_lxc_do_info` | `false` | No | Run the `info` section (tag `info`) |
| `proxmox_lxc_do_create` | `false` | No | Run the `create` section (tag `create`) |
| `proxmox_lxc_do_clone` | `false` | No | Run the `clone` section (tag `clone`) |
| `proxmox_lxc_do_lifecycle` | `false` | No | Run the `lifecycle` section (tag `lifecycle`) |
| `proxmox_lxc_do_template` | `false` | No | Run the `template` section (tag `template`) |
| `proxmox_lxc_vmid` | `""` | No | VMID for a new container; empty lets PVE assign the next free one |
| `proxmox_lxc_node` | `""` | If creating | PVE node to create the container on |
| `proxmox_lxc_hostname` | `""` | If creating | Container hostname |
| `proxmox_lxc_ostemplate` | `""` | If creating | Template volume, e.g. `local:vztmpl/rockylinux-9-default_20240523_amd64.tar.xz` |
| `proxmox_lxc_password` | `"{{ vault_proxmox_lxc_password \| default(omit) }}"` | No | Root password; defaults to `vault_proxmox_lxc_password` when defined |
| `proxmox_lxc_pool` | `""` | No | Resource pool to place the container in |
| `proxmox_lxc_storage` | `"local-lvm"` | No | Storage for the root filesystem |
| `proxmox_lxc_disk_size` | `"8"` | No | Root disk size in GiB (no suffix); combined with storage into `storage:size` |
| `proxmox_lxc_cores` | `2` | No | CPU cores |
| `proxmox_lxc_memory` | `1024` | No | Memory in MiB |
| `proxmox_lxc_swap` | `512` | No | Swap in MiB |
| `proxmox_lxc_net0` | `"name=eth0,bridge=vmbr0,ip=dhcp"` | No | First NIC definition |
| `proxmox_lxc_nameserver` | `""` | No | DNS server for the container |
| `proxmox_lxc_searchdomain` | `""` | No | DNS search domain |
| `proxmox_lxc_pubkey` | `""` | No | SSH public key content (not a path) for root |
| `proxmox_lxc_unprivileged` | `true` | No | Create an unprivileged container |
| `proxmox_lxc_features` | `[]` | No | Feature flags, e.g. `["nesting=1", "keyctl=1"]` |
| `proxmox_lxc_onboot` | `true` | No | Start the container on node boot |
| `proxmox_lxc_tags` | `[]` | No | Tags to apply |
| `proxmox_lxc_create_timeout` | `300` | No | Seconds to wait for creation |
| `proxmox_lxc_start_after_create` | `true` | No | Start the container after creating it |
| `proxmox_lxc_clone_source_name` | `""` | If cloning | Name of the container to clone |
| `proxmox_lxc_clone_node` | *(source node)* | No | Node for the clone; empty uses the source's node |
| `proxmox_lxc_clone_vmid` | `""` | No | VMID for the clone; empty lets PVE assign one |
| `proxmox_lxc_clone_hostname` | `""` | If cloning | Hostname for the new container |
| `proxmox_lxc_clone_type` | `"opportunistic"` | No | `full`, `linked` (needs a template source), or `opportunistic` |
| `proxmox_lxc_clone_storage` | `""` | If a full clone | Target storage; empty leaves it to PVE |
| `proxmox_lxc_clone_timeout` | `300` | No | Seconds to wait for the clone |
| `proxmox_lxc_clone_start_after` | `true` | No | Start the clone after it is created |
| `proxmox_lxc_target_name` | `""` | If lifecycle | Existing container to start/stop/restart/destroy, looked up by name |
| `proxmox_lxc_lifecycle_state` | `""` | If lifecycle | `started`, `stopped`, `restarted`, or `absent` |
| `proxmox_lxc_force_stop` | `false` | No | With `stopped`, power off instead of a graceful shutdown |
| `proxmox_lxc_purge` | `false` | No | With `absent`, also remove from backup, replication, and HA config |
| `proxmox_lxc_destroy_unreferenced_disks` | `true` | No | With `absent`, destroy disks not referenced by the config |
| `proxmox_lxc_lifecycle_timeout` | `300` | No | Seconds to wait for the lifecycle change |
| `proxmox_lxc_template_source_name` | `""` | If templating | Stopped container to convert into a template |
| `proxmox_lxc_template_force` | `true` | No | Stop the container first if it is running |


### Connection

Every `proxmox_*` role in this collection uses the same `proxmox_api_*`
vars — see `README.md`. Set them once for the whole
play/inventory; a VM/CT lives on one node in one cluster, so there's
nothing role-specific to override here.

The roles hand these to the `community.proxmox` modules through `module_defaults`
(see `README.md`), so tasks here don't repeat them.

```yaml
proxmox_api_host: "pve2.example.com"   # any PVE node or the cluster VIP
```

### Execution gates

| Gate variable | Default | Tag | What it runs |
|---|---|---|---|
| `proxmox_lxc_do_info` | `false` | `info` | Cluster/node/storage lookup |
| `proxmox_lxc_do_create` | `false` | `create` | Create a new container from a vztmpl |
| `proxmox_lxc_do_clone` | `false` | `clone` | Clone + start a new container from an existing one |
| `proxmox_lxc_do_lifecycle` | `false` | `lifecycle` | Start/stop/restart/destroy an existing container |
| `proxmox_lxc_do_template` | `false` | `template` | Convert a container into a template |

### info

No variables required. Sets facts:

| Fact | Source |
|---|---|
| `proxmox_lxc_cluster_status` | `community.proxmox.proxmox_cluster_status_info` |
| `proxmox_lxc_node_facts` | `community.proxmox.proxmox_node_info` |
| `proxmox_lxc_storage_facts` | `community.proxmox.proxmox_storage_info` |

### create

```yaml
proxmox_lxc_node:       "pve2"
proxmox_lxc_hostname:   "newct01"
proxmox_lxc_ostemplate: "local:vztmpl/rockylinux-9-default_20240523_amd64.tar.xz"
proxmox_lxc_password:   "{{ vault_proxmox_lxc_password }}"
proxmox_lxc_storage:    "local-lvm"
proxmox_lxc_disk_size:  "8"      # GiB, no suffix
proxmox_lxc_cores:      2
proxmox_lxc_memory:     1024
proxmox_lxc_swap:       512
proxmox_lxc_net0:       "name=eth0,bridge=vmbr0,ip=dhcp"
proxmox_lxc_pubkey:     "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
proxmox_lxc_unprivileged: true
proxmox_lxc_features:   ["nesting=1"]
proxmox_lxc_tags:       ["lab"]
```

`proxmox_lxc_vmid` defaults to `""`, letting PVE assign the next free VMID.
`proxmox_lxc_storage`/`proxmox_lxc_disk_size` are combined into the PVE
`disk` string (`storage:size`) — the underlying module treats `disk` and
`storage` as alternate ways to target the rootfs, so this role only ever
sends `disk`.

### clone

```yaml
proxmox_lxc_clone_source_name: "rocky9-lxc-template"   # looked up by name
proxmox_lxc_clone_hostname:    "newct02"
proxmox_lxc_clone_node:        ""       # "" = same node as the source
proxmox_lxc_clone_type:        "opportunistic"   # full | linked | opportunistic
proxmox_lxc_clone_storage:     "local-lvm"        # required for a full clone
```

`clone_type: linked` only works when the source container is itself a PVE
template. `opportunistic` (the module default) does a linked clone when the
source is a template and a full clone otherwise.

### lifecycle

```yaml
proxmox_lxc_target_name:     "oldct01"       # looked up by name via proxmox_vm_info
proxmox_lxc_lifecycle_state: "absent"        # started | stopped | restarted | absent
proxmox_lxc_force_stop:      false           # stopped only — skip graceful shutdown
proxmox_lxc_purge:           false           # absent only — remove from backup/replication/HA
proxmox_lxc_destroy_unreferenced_disks: true # absent only
```

Fails if zero or multiple containers match `proxmox_lxc_target_name` — same
discovery pattern as `proxmox_vm`.

### template

```yaml
proxmox_lxc_template_source_name: "rocky9-lxc-golden"
proxmox_lxc_template_force:       true   # stop the container first if running
```


Tags
----

| Tag | What it runs |
|-----|-------------|
| `proxmox_lxc` | Entire role |
| `info` | Cluster/node/storage lookup |
| `create` | Create from vztmpl |
| `clone` | Clone + start |
| `lifecycle` | Start/stop/restart/destroy |
| `template` | Convert to template |


Example Playbook — create a container from a template
-------------------------------------------------------

```yaml
- name: Provision a new LXC container
  hosts: localhost
  gather_facts: false
  roles:
    - role: mgcdrd.proxmox.proxmox_lxc
      vars:
        proxmox_api_host:         "pve2.example.com"
        proxmox_api_token_id:     "{{ vault_proxmox_api_token_id }}"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_lxc_do_create: true
        proxmox_lxc_node:       "pve2"
        proxmox_lxc_hostname:   "newct01"
        proxmox_lxc_ostemplate: "local:vztmpl/rockylinux-9-default_20240523_amd64.tar.xz"
        proxmox_lxc_password:   "{{ vault_proxmox_lxc_password }}"
        proxmox_lxc_storage:    "local-lvm"
        proxmox_lxc_disk_size:  "8"
        proxmox_lxc_cores:      2
        proxmox_lxc_memory:     1024
        proxmox_lxc_pubkey:     "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
```

Example Playbook — clone a container
--------------------------------------

```yaml
- name: Clone an LXC container from a template
  hosts: localhost
  gather_facts: false
  roles:
    - role: mgcdrd.proxmox.proxmox_lxc
      vars:
        proxmox_api_host:         "pve2.example.com"
        proxmox_api_token_id:     "{{ vault_proxmox_api_token_id }}"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_lxc_do_clone: true
        proxmox_lxc_clone_source_name: "rocky9-lxc-template"
        proxmox_lxc_clone_hostname:    "newct02"
        proxmox_lxc_clone_type:        "linked"
```

Example Playbook — tear down a container
--------------------------------------------

```yaml
- name: Remove a decommissioned container
  hosts: localhost
  gather_facts: false
  roles:
    - role: mgcdrd.proxmox.proxmox_lxc
      vars:
        proxmox_api_host:         "pve2.example.com"
        proxmox_api_token_id:     "{{ vault_proxmox_api_token_id }}"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_lxc_do_lifecycle: true
        proxmox_lxc_target_name: "oldct01"
        proxmox_lxc_lifecycle_state: "absent"
```


Notes
-----

- **`delegate_to: localhost`**: every task in this role talks to the PVE API,
  not the inventory host — run with `hosts: localhost`, as in the examples.
- **VMID uniqueness**: if multiple containers share a name across PVE nodes,
  `clone.yml`/`lifecycle.yml`/`to_template.yml` fail rather than guess.
- **`netif` is a raw string per interface** (`net0: "name=eth0,..."`), the
  same convention `proxmox_vm` uses for `net0` — not a nested dict of
  individual keys.
- **`startup` (boot/shutdown ordering) is deliberately not exposed** by this
  role — the upstream module accepts it as a list of strings but its exact
  per-element format isn't documented anywhere in the module's own docs.
  Set it directly in PVE's UI/API if you need it, or open an issue if you
  need it added here once the format is confirmed.
- **No handlers**: every module call is idempotent against PVE's own state.
