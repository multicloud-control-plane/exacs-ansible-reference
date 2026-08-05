# Database lifecycle operations

Each mutating operation requires a reviewed operation input and an explicit
`operation_execution_mode`: run `precheck` first and use `execute` only after
reviewing its evidence. Almost every playbook uses OCI APIs through
`oracle.oci` and needs no access to a database node. Three exceptions are
documented below: two that use the OCI CLI, and the database bounce, which uses
`dbaascli` over SSH.

The **Proven** column is the honest state of each operation against a real VM
Cluster, not an aspiration. `full` means `precheck`, `apply` and `verify` have
all run; `precheck` means only the non-mutating stage has; `none` means the
operation has never executed at all. Treat anything other than `full` as code
to review before you trust it.

| Lifecycle stage | Operation | Master playbook | Proven | Effect |
| --- | --- | --- | --- | --- |
| Day 1 | Database Home create | `exacs-db-home-create.yml` | full | Creates one approved Database Home on an existing VM Cluster. |
| Day 1 | CDB create | `exacs-cdb-create.yml` | full | Creates one CDB and its initial PDB in an approved Database Home. |
| Day 1 | PDB create | `exacs-pdb-create.yml` | full | Creates one additional PDB in an approved CDB. |
| Day 2 | CDB out-of-place patch | `exacs-database-out-of-place-patch.yml` | full | Moves one CDB to an approved compatible Database Home. |
| Day 2 | Move without datapatch | `exacs-database-move-skip-datapatch.yml` | full | Moves one CDB to a higher Database Home, leaving datapatch outstanding. |
| Day 2 | Run datapatch | `exacs-database-run-datapatch.yml` | full | Runs datapatch on one approved CDB, and optionally on named PDBs. |
| Day 2 | Database bounce | `exacs-database-bounce.yml` | **none** | Restarts one approved CDB on its node. Needs SSH. |
| Day 2 | PDB restart | `exacs-pdb-restart.yml` | full | Stops and starts one approved PDB. |
| Day 2 | DB node restart | `exacs-db-node-restart.yml` | **precheck** | Stops and starts one approved DB node. |
| Day 2 | VM Cluster scale | `exacs-vm-cluster-scale.yml` | full | Scales one approved Cloud VM Cluster up or down, including to 0 OCPU. |
| Day 2 | Password rotate | `exacs-database-password-rotate.yml` | **none** | Sets a new administrator password on one approved CDB. |
| Evidence | Resource discovery | `exacs-resource-discovery.yml` | full | Reports the VM Cluster's current OCPU capacity, plus the Database Homes, CDBs, and PDBs owned by a project and environment. |

Two entries deserve their caveat spelled out. **Database bounce has never run,
not even its precheck**, because the validation environment has no network path
to the cluster's nodes; it is the only operation in that position, and the only
one that acts on a host. **DB node restart** has passed its precheck but has
never been executed, because doing so restarts a real cluster node.

**Scaling down is allowed**, and a target of 0 OCPU is legal. The precheck names
the consequence before you approve the run: at 0 OCPU every database on the
cluster is left without CPU and stops serving connections until it is scaled up
again. The source count in the manifest must match the cluster's current count,
and a target equal to the source is refused as a no-op.

The supplied examples contain fictional identifiers. Copy an example into a
protected customer repository, replace its values, and retain the precheck and
post-operation output as evidence.

For example, to restart one PDB after reviewing its precheck:

```bash
ansible-playbook ansible/playbooks/exacs-pdb-restart.yml \
  -e operation_file=$PWD/ansible/pdb-restart.yml \
  -e operation_execution_mode=precheck
```

Run the same command with `operation_execution_mode=execute` only after the
customer has approved the reported boundary.

## The three execution layers

Every operation resolves identity the same way and passes through the same
`precheck` and `execute` gate. What differs is only where the change is made,
and each layer is used only after the one above it has been ruled out.

| Layer | Used when | Operations |
| --- | --- | --- |
| `oracle.oci` module | The collection has a module | All but the two below |
| OCI CLI | OCI has the API, the collection has no module | Move without datapatch, run datapatch |
| `dbaascli` over SSH | OCI has no API at all | Database bounce |

The bounce is the clearest case. OCI can start and stop a *pluggable* database
and it can restart a whole DB node, but there is no API operation anywhere that
starts or stops a *container* database. Restarting one is therefore only
possible on the node.

Note what does not change when the layer drops: identity still comes from the
ownership tags through the shared resolvers, and `verify.yml` still reads the
result back through the OCI API. A change made on a host does not become
invisible to the control plane.

To add your own host operation, copy `common/exacs/host/run-dbaascli.yml` and
change the arguments. It discovers the node from OCI facts, so no inventory
file is needed.

### Reaching a node

A VM Cluster subnet normally forbids public IPs, so the helper takes an
optional jump host. One setting covers all three cases:

| Situation | Setting |
| --- | --- |
| Runner already routed to the VCN | Leave the bastion values empty |
| Your own jump host | Its address, and the SSH user on it |
| OCI Bastion service | The regional endpoint, with the **session OCID as the user** |

The third needs an **SSH port forwarding** session, which is consumed exactly
like a jump host. It cannot be a managed SSH session: those require Oracle
Cloud Agent with the Bastion plugin on the target, and a VM Cluster node is not
a compute instance that runs it. Two consequences worth knowing before the
first run:

- A port forwarding session is bound to one address and port. Point it at the
  node this operation resolves from the ownership tags, port 22, or the bastion
  refuses the connection.
- `ssh` does not pass command-line options to a jump host, so the private key
  in the manifest authenticates the node only. Give the bastion leg its own
  `IdentityFile` in the runner's `~/.ssh/config`.

Sessions expire, three hours at most, so create one just before the run;
`oracle.oci` has an `oci_bastion_session` module if you want to automate that
step too.

## Running datapatch, and why it uses the OCI CLI

Every other operation calls an `oracle.oci` module. `exacs-database-run-datapatch`
is the exception: OCI exposes `RunDataPatch` in its API and CLI, but the
collection has no module for it, so the `apply` task calls the CLI directly.
That is the fallback this reference allows when the collection lags the API,
and it needs the OCI CLI installed on the runner.

Only the mutating call changes. Identity is still resolved from ownership tags
by the shared resolvers, the `precheck` and `verify` stages still read OCI
facts through the collection, and the same `precheck` and `execute` evidence
gate applies. The CLI is given the same authentication the modules use, so one
runner configuration drives both.

The `precheck` here is stronger than in the other operations. It does not only
assert boundary conditions: it calls the API with `--action PRECHECK`, so OCI
itself reports whether the datapatch would apply.

## When a CDB or PDB creation fails

Those two operations pass an administrator password, so their creation task
hides its output. They report the failure anyway: a `rescue` prints the error
text returned by OCI, for example

```text
Creating CDB FINCDB failed: query param compartmentId cannot be null.
```

That text comes from the API response and never contains the password, so the
run stays diagnosable without lowering the password protection.

The pattern is the same irrespective of whether the underlying Exadata
Infrastructure and VM Cluster were provisioned from Azure, AWS, or GCP, because
every request goes to OCI.

Deletions and Data Guard are deliberately not implemented. Removing a database
is the one action this reference will not do for you.
