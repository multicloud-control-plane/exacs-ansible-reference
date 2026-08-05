# ExaCS Ansible Database Lifecycle Reference

A customer-ready reference implementation for Oracle Exadata Database Service
on Dedicated Infrastructure (ExaCS).

It manages the ExaCS database lifecycle through the OCI Control Plane and the
OCI Ansible Collection: creating Database Homes, Container Databases (CDBs) and
Pluggable Databases (PDBs), and a reviewed set of Day 2 operations on them —
patching, restarts, and scaling the VM Cluster. See
[`docs/day2-operations.md`](docs/day2-operations.md) for the full list, and for
how far each one has been proven against a real VM Cluster.

Using it requires no prior experience with the OCI Ansible Collection. Every
operation is one command. All but one work purely through OCI APIs and need
no access to a database node.

It deliberately does not provision network, Exadata Infrastructure, or a VM
Cluster. Those are prerequisites created from the relevant Azure, AWS, or GCP
control plane. Once a supported ExaCS VM Cluster exists, this pattern is the
same for every hyperscaler because database lifecycle requests go to OCI.

- [Customer journey](#customer-journey)
- [Safety and scope](#safety-and-scope)
- [Repository layout](#repository-layout)
- [Quick start](#quick-start)
- [Pinned source versions](#pinned-source-versions)
- [Further documentation](#further-documentation)
- [License](#license)

## Customer journey

1. Declare a Database Home, CDB, or PDB in an approved operation manifest.
2. Run its master playbook in `precheck` mode and review the observed OCI facts.
3. Run it in `execute` mode to create the resource, then retain verification
   evidence.
4. Use resource discovery to view the OCI state owned by the project and
   environment.
5. For a patch or restart, use the same precheck / execute evidence gate.

## Safety and scope

This is a reference implementation to review and adapt, not a package to
clone and run unmodified against a production tenancy. Read every playbook
you plan to use, copy its manifest into your own protected repository, and
check its `Proven` status in [`docs/day2-operations.md`](docs/day2-operations.md)
before the first `execute` run. See
[`docs/production-considerations.md`](docs/production-considerations.md) for
what to harden — tag governance and separation of duties — before operating
this for consumers who did not write it.

This repository contains no customer OCIDs, API keys, PEM files, passwords, or
production names. Every identifier in an example is fictional and must be
replaced before use. Passwords are read from the runner environment and must
never be committed.

Almost every operation uses OCI Control Plane APIs only. The exception is the
database bounce, which OCI does not expose at all and which therefore runs
`dbaascli` on a node over SSH. It is opt-in: no other operation needs host
access.

Two operations act on the cluster rather than on a database. Restarting a DB
node and scaling the VM Cluster's OCPUs both affect every database running
there, so each one names that consequence in its precheck before you approve
it. Scaling accepts a lower target, including 0 OCPU, which leaves the
databases without CPU until the cluster is scaled back up.

## Repository layout

```text
ansible/               Day 1 and Day 2 masters, reusable common tasks, and examples
ansible/inventory/     Optional OCI dynamic inventory, for the one operation using SSH
docs/                  Architecture, authentication, inventory, and operation guide
```

## Quick start

```bash
python3.12 -m venv .venv
. .venv/bin/activate
pip install ansible==14.2.0 oci==2.183.0
ansible-galaxy collection install -r ansible/requirements.yml
```

Then follow [`docs/getting-started.md`](docs/getting-started.md) to verify the
install, and [`docs/customer-guide.md`](docs/customer-guide.md) for the first
`precheck` run.

## Pinned source versions

- Python `3.12`
- Ansible `14.2.0` (`ansible-core` 2.21)
- OCI Python SDK `2.183.0`
- OCI Ansible Collection `5.5.0`

## Further documentation

Read [`docs/architecture.md`](docs/architecture.md) for the OCI Control Plane
boundary, [`docs/resource-identity.md`](docs/resource-identity.md) for how
resources are identified and why,
[`docs/authentication.md`](docs/authentication.md) for the supported
runner authentication choices, [`docs/getting-started.md`](docs/getting-started.md)
for installation, [`docs/customer-guide.md`](docs/customer-guide.md) for
the full lifecycle, [`docs/day2-operations.md`](docs/day2-operations.md)
for the catalog of available operations, and
[`docs/production-considerations.md`](docs/production-considerations.md) for
what changes before this runs as an operated platform rather than a
reference.

## License

Released under the [Universal Permissive License 1.0](LICENSE).
