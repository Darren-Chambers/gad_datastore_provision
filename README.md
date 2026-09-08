# gad_datastore_provision

One AAP survey, one job: create an LDEV on a Hitachi VSP, present it to
an ESX cluster, protect it with GAD, present the S-VOL on the secondary,
rescan, and create the VMFS datastore. Successor to
[gad_to_datastore](https://github.com/Darren-Chambers/gad_to_datastore),
which only did the last step and needed the volume and pair created in
VSP360 first.

Nothing about the ESX cluster's storage layout is configured. It is
discovered by matching host WWPNs (from vCenter) against the WWNs
registered in host groups (from the arrays).

---

## Workflow

```
Nightly:  update_aap_survey.yml  ->  rebuilds the survey from live data
            - GAD pairs from the vault filenames
            - clusters that are GAD-capable on a pair (all hosts have FC host
              groups on BOTH arrays, secondary ones in a VSM resource group)
            - pools on every array with free capacity in the label

Operator: launches "Provision GAD Datastore", fills in the survey,
          leaves Confirm blank  ->  PLAN: full discovery, prints exactly
          what would be created, changes nothing, job ends green

Operator: re-launches with Confirm = CONFIRM  ->  PROVISION:
            1. hv_ldev         P-VOL in chosen pool, named after the datastore
            2. hv_hg           P-VOL presented to every matched primary host
                               group at a LUN free on every host group both sides
            3. hv_gad          pair created; S-VOL created in chosen secondary
                               pool and presented to matched secondary host groups
            4. hv_ldev_facts   canonical NAA read back from the array
            5. vmware          rescan every host; require every host to see the
                               NAA and require it to be unclaimed
            6. vmware          VMFS datastore on the verified device; rescan others
```

If a storage step fails before the GAD pair exists, the P-VOL is deleted
(force) and the failure is reported. If it fails after, nothing is
deleted and the message says what to inspect.

---

## Discovery

```
vCenter  cluster -> hosts -> FC HBA WWPNs         (vmware_host_vmhba_info)
Array    host groups -> registered WWNs, LUN paths (hv_hg_facts query: [wwns, ldevs])

host group belongs to cluster  <=>  any of its WWNs is a WWPN of a host in the cluster
cluster capable on a pair      <=>  every host matched >=1 host group on the primary
                                    AND >=1 on the secondary
                                    AND no matched secondary host group is in RG 0
```

The survey labels record what was discovered at refresh time. The
provisioning run repeats the discovery and validates against live data;
labels are never trusted. A cluster that is not fully presented on both
arrays does not appear in the survey, and is a hard failure if it is
somehow selected. This is the guard against a datastore some hosts
cannot see.

Also resolved automatically at run time:

| Item | How |
| --- | --- |
| Quorum disk | `hv_quorum_disk_facts` on the primary: the single `NORMAL` quorum whose `remote_serial_number` is the secondary. Zero or several -> fail; set `quorum_disk_id` in `vars/site_config.yml` to pin one. |
| LUN | Lowest number not used by any matched host group on either array, so every host sees the device at the same LUN. Override with `lun_id`. |
| Copy group | `AAP_<cluster>` (sanitised, 31 chars). One copy group per cluster. |
| Copy pair | The datastore name. |
| S-VOL ID | Left to `hv_gad` (same ID as P-VOL if free on the secondary). |

Name collisions checked before the gate: datastore name in vCenter, LDEV
name on the primary.

---

## Prerequisites the playbook does NOT do

`hv_gad` assumes the GAD infrastructure exists (it checks, it does not
build): remote storage registration between the arrays, remote paths,
the quorum disk, and FC host groups with valid WWNs on both arrays for
every host. Secondary host groups must be in a VSM resource group, not
RG 0 (module refuses otherwise; the survey refresh reports it).

FC only. Hosts with no FC WWPNs are reported and their cluster excluded.

---

## Project structure

```
gad_datastore_provision/
├── provision_gad_datastore.yml     # the job
├── update_aap_survey.yml           # nightly survey rebuild
├── tasks/
│   ├── load_site.yml               # site config + AAP/vCenter vault
│   ├── load_pair.yml               # pair from ansible_vault_storage_<serial>.yml
│   ├── discover_cluster.yml        # cluster hosts + WWPNs
│   ├── fetch_array_hostgroups.yml  # host groups w/ WWNs + LUN paths, per array
│   ├── match_hostgroups.yml        # WWN join, pure data
│   ├── survey_cluster.yml          # refresh: wraps discover_cluster, skips on error
│   ├── survey_pair.yml             # refresh: pools + host groups for one pair
│   └── survey_pair_cluster.yml     # refresh: capability test for one pair/cluster
├── vars/site_config.yml.example
├── ansible_vault_vars/
│   ├── ansible_vault.yml.example                   # AAP + vCenter creds
│   └── ansible_vault_storage_SERIAL.yml.example    # one per GAD pair
└── collections/requirements.yml
```

---

## Setup

### 1. Site config

```
cp vars/site_config.yml.example vars/site_config.yml
```

Gitignored. vCenter, AAP URL, provisioning defaults (`ldev_capacity_saving`,
`vmfs_version`, `gad_copy_pace`), optional `quorum_disk_id` / `lun_id` pins.

### 2. Vault files

```
cp ansible_vault_vars/ansible_vault.yml.example ansible_vault_vars/ansible_vault.yml
ansible-vault encrypt ansible_vault_vars/ansible_vault.yml
```

One file per GAD pair, **named after the primary serial** — the survey
refresh discovers pairs from these filenames:

```
cp ansible_vault_vars/ansible_vault_storage_SERIAL.yml.example \
   ansible_vault_vars/ansible_vault_storage_840130.yml
ansible-vault encrypt ansible_vault_vars/ansible_vault_storage_840130.yml
```

Layout is the Hitachi `vspone_block` example convention (`storage_serial`,
`storage_address`, `vault_storage_username/secret`, and the
`secondary_*` equivalents). Encrypted files are gitignored; commit only
the `.example` files. AAP decrypts them with the Vault Credential.

### 3. Job templates

| | Provision GAD Datastore | Provision GAD Datastore — Survey Refresh |
| --- | --- | --- |
| Playbook | `provision_gad_datastore.yml` | `update_aap_survey.yml` |
| Inventory | localhost | localhost |
| Credentials | Vault Credential | Vault Credential |
| Extra vars | — | `aap_job_template_id: <ID of the provision template>` |
| Survey | enabled (built by refresh) | — |
| Schedule | — | nightly |

Run the refresh once by hand to create the survey.

### 4. Execution environment

`hitachivantara.vspone_block >= 4.8` and `community.vmware < 7`
(`vmware_cluster_info` was removed in 7.0; migrating to `vmware.vmware`
is a straightforward swap when needed).

---

## Survey

| Question | Variable | Source |
| --- | --- | --- |
| GAD Pair | `gad_pair` | vault filenames, label `840130 -> 840183` |
| ESX Cluster | `esx_cluster` | capable clusters, label `NAME [pair, ...]` |
| Primary Pool | `primary_pool` | `serial \| pool N \| name \| X TiB free` |
| Secondary Pool | `secondary_pool` | same list; must be on the secondary |
| Datastore Name | `datastore_name` | 1–32 chars `[A-Za-z0-9_-]` |
| Size (GB) | `size_gb` | integer ≥ 10 |
| Confirm | `confirm` | blank = plan, `CONFIRM` = provision |

Pool lists are not filtered per array (surveys have no dependent
questions); picking a pool on the wrong array fails before the gate.

---

## Notes

- Preferred path / ALUA is not set on the secondary host groups
  (`enable_preferred_path: false`). For a stretched cluster with hosts at
  both sites, revisit this.
- Primary is always the array the vault file is named for. To provision
  with the pair the other way round, add a second vault file named for
  the other serial.
- `hv_gad` vs `hv_vsp_one_gad`: the docs steer VSP One B85 to the latter.
  B28s worked with `hv_gad` on 3.1; if 4.8 objects, the spec is the same
  shape and the swap is one module name.
