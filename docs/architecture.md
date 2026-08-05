# Architecture and ownership boundaries

```text
Azure, AWS, or GCP control plane
  → network, Exadata Infrastructure, VM Cluster

OCI Control Plane + OCI Ansible Collection
  → Database Home
  → CDB and initial PDB
  → additional PDB
  → Day 2 on those databases: out-of-place patch, datapatch, restarts
  → Day 2 on the cluster: DB node restart, OCPU scale
```

The cloud-specific platform creates the infrastructure boundary. From the
existing VM Cluster onwards, this repository uses the OCI Control Plane, so the
same Ansible pattern applies to ExaCS on Azure, AWS, and GCP.

Each lifecycle action is a master playbook with reusable `precheck.yml`,
`apply.yml`, and `verify.yml` common tasks. The master runs on `localhost` and
uses `oracle.oci` APIs. Only the database bounce needs SSH to a node, because
OCI exposes no API for it.
`precheck` is non-mutating. `execute` is explicit and performs the API call,
waits for OCI, and reads facts again to verify the result.

The operation manifest is the requested business intent. It carries a
`resource_key` plus the ownership tags `exacs_project` and
`exacs_environment`, and never an OCID for anything the automation creates.
At runtime the playbook scopes OCI facts by resource type, compartment, and
VM Cluster or CDB parent, filters on those three tags, and requires exactly
one match before using the resolved OCID. The same rule applies to Database
Homes, CDBs, and PDBs alike; display names are cosmetic, because OCI does not
enforce them as unique. See [resource-identity.md](resource-identity.md) for
why this model was chosen and what its limits are.

OCI is therefore the observed state. There is no separate persisted OCID
registry and no competing database inventory. The read-only resource-discovery
master is the evidence view of that observed state.
