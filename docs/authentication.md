# Authentication choices

## Choosing the mode

Every master playbook takes its authentication mode from a single variable,
`exacs_auth_type`. Set it once, in either of two ways:

```bash
# Environment variable, applies to every playbook in the shell session
export OCI_ANSIBLE_AUTH_TYPE=api_key

# Or per run, on the command line
ansible-playbook ... -e exacs_auth_type=api_key
```

When neither is given, the playbooks assume `instance_principal`.

| Mode | Use it when | Needs an SDK config file |
| --- | --- | --- |
| `instance_principal` | The runner is an OCI compute instance or OKE node | No |
| `api_key` | The runner is anywhere else | Yes |

Instance Principal is the recommended mode because it avoids a static private
key. It is only available from inside OCI, so a runner hosted in Azure, AWS,
GCP, or on-premises must use `api_key` — that is the expected mode for a
multicloud control plane.

## The policy the automation needs

Authenticating and being allowed to do something are separate problems, and the
second one is easy to misread: OCI answers a request for a resource you have no
permission to see with a "not found", so a missing policy looks exactly like a
missing database. If a `precheck` cannot find a resource you know exists, check
the policy before you check the manifest.

The principal is a dynamic group for `instance_principal`, or a group
containing the automation user for `api_key`. Either way it needs these
statements, scoped to the compartment holding the databases:

```text
Allow dynamic-group exacs-automation to manage db-homes            in compartment <compartment>
Allow dynamic-group exacs-automation to manage databases           in compartment <compartment>
Allow dynamic-group exacs-automation to manage pluggable-databases in compartment <compartment>
Allow dynamic-group exacs-automation to manage db-nodes            in compartment <compartment>
Allow dynamic-group exacs-automation to use    cloud-vmclusters    in compartment <compartment>
Allow dynamic-group exacs-automation to read   cloud-exadata-infrastructures in compartment <compartment>
```

Three things about that list are worth knowing rather than discovering:

- The resource type is `cloud-vmclusters`, with no hyphen in the middle. It is
  the one name in the set that does not follow the pattern of the others, and a
  policy with `cloud-vm-clusters` in it is accepted and then matches nothing.
- `use` is enough for the VM Cluster because scaling OCPUs is an update, not a
  creation. This reference never creates or destroys a VM Cluster, so `manage`
  would be granting the automation a power it has no operation for.
- `manage` on `db-nodes` is required, and is not symmetrical with the point
  above: a node restart is a power action, and OCI puts power actions in
  `manage` rather than in `use`.

The three `manage` statements on databases also carry delete. This reference
implements no deletion at all, so if your tenancy's review requires it, deny
that explicitly rather than dropping to `use` — `use` cannot create, and
creation is most of what this reference does:

```text
Allow dynamic-group exacs-automation to manage databases in compartment <compartment>
 where request.permission != 'DATABASE_DELETE'
```

These statements are a starting point derived from Oracle's policy reference
for Exadata Database Service on Dedicated Infrastructure, not a set proven
against a tenancy. Confirm them with your own `precheck` runs, which is the
cheapest possible test: a precheck reads and asserts, and changes nothing.

## OCI Ansible Collection

The OCI Ansible Collection does not read `OCI_AUTH`. Set
`OCI_ANSIBLE_AUTH_TYPE` for the playbook separately.

### Recommended: Instance Principal

```bash
export OCI_ANSIBLE_AUTH_TYPE=instance_principal
```

### Compatible option: API key

Keep the private PEM outside the repository. OCI Ansible reads a standard SDK
profile through `OCI_CONFIG_FILE` and `OCI_CONFIG_PROFILE`. `oci setup config`
writes that profile to `~/.oci/config` by default, and that default is fine to
use directly:

```bash
export OCI_CONFIG_FILE=~/.oci/config
export OCI_CONFIG_PROFILE=EXACS_AUTOMATION
export OCI_ANSIBLE_AUTH_TYPE=api_key
```

Move the config file and PEM to a more restricted path if your runner's
threat model calls for it; `OCI_CONFIG_FILE` accepts any path.

For GitHub Actions, protect any API-key secret with an Environment and only
materialize it for an approved job. A workload identity or Instance Principal
remains preferable because it avoids a static PEM.
