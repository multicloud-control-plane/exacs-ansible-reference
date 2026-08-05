# Resource discovery and host inventory

The database lifecycle masters run locally and use OCI API facts to discover
Database Homes, CDBs, and PDBs. This is intentionally different from Ansible
host inventory: API operations do not need a database-node SSH target.

`exacs-resource-discovery.yml` is the resource inventory master. It queries
OCI every run and filters results by `exacs_project` and `exacs_environment`.
It is read-only and does not save resolved OCIDs in Git, a file, or a registry.

The optional `ansible/inventory/exacs-dbhosts.oci.example.yml` uses the
`oracle.oci.oci` dynamic inventory plugin with `fetch_db_hosts: true`. Adopt it
when a separately approved operation needs host connectivity, such as
`dbaascli`. Provide the correct region, compartment, hostname format, SSH
identity, and network route for the customer environment. The plugin uses the
same OCI authentication context as the collection; no credentials belong in the
inventory file.

For a collection gap, a common task may call OCI CLI locally from the runner;
it still works through the OCI Control Plane. A `dbaascli` task is different: it
runs on a discovered database host and requires the additional secure-connectivity
controls above. In both cases, the master remains the orchestrator and must
keep the same precheck, explicit execute gate, wait, verification, and evidence
pattern.
