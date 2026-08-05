# How resources are identified, and why

Every operation in this repository has to answer one question before it can do
anything: *which* Database Home, CDB, or PDB does this manifest mean? This
document records how that question is answered, what else was considered, and
what the trade-offs are. It exists because the answer shapes everything else —
changing it later, once a customer has databases in production, is expensive.

## The rule

A resource is identified by three ownership tags applied at creation:

| Tag | Meaning |
| --- | --- |
| `exacs_project` | Which project owns the resource |
| `exacs_environment` | Which environment it belongs to, for example `nonprod` |
| `exacs_resource_key` | A stable key unique within that project and environment |

Every operation lists resources from OCI, filters on those three tags, and
asserts that **exactly one** resource matches:

```yaml
| selectattr('freeform_tags.exacs_project',      'equalto', exacs_project)
| selectattr('freeform_tags.exacs_environment',  'equalto', exacs_environment)
| selectattr('freeform_tags.exacs_resource_key', 'equalto', <key>)
| list | length == 1
```

That `length == 1` assertion is the safety invariant. Zero matches means the
manifest points at something that does not exist; more than one means the
ownership model has been broken. Both refuse to proceed.

The rule applies uniformly to Database Homes, CDBs, and PDBs. There is no
resource type that is identified some other way.

## Why tags, and not the alternatives

### Not a state file

The obvious alternative is what Terraform does: record the OCID of everything
created in a state file, and read that file on the next run. It was rejected
because a state file is a second source of truth that has to be stored,
shared, locked, and backed up. If it is lost or drifts from reality, the
automation is stranded. Tags keep OCI itself as the only source of truth, so a
run works from any machine or pipeline with no shared backend.

### Not OCIDs in the manifest

The other alternative is to write the OCIDs directly into the manifest. That is
simple and unambiguous, and it is still how this repository handles the two
boundary resources it does not create — the compartment and the VM Cluster.

It was rejected for the resources the automation *does* create, for two
reasons. An OCID is opaque, so a manifest full of them cannot be reviewed by a
human before an `execute` run, which defeats the precheck gate. And the OCID
does not exist until the resource has been created, so the manifest could not
be written up front.

### Not display names

Display names look like a natural key, and an earlier version of this
repository used them for Database Homes. That was wrong: OCI does not enforce
display names as unique, so two Database Homes can carry the same name and the
`length == 1` assertion becomes a coin toss. Display names are now cosmetic
and are never used to resolve anything.

This is the general industry position too. HashiCorp's Well-Architected
Framework and the AWS tagging best-practices whitepaper both recommend that
automation select resources by tag rather than by identifier or name, on the
grounds that tags survive recreation and are portable between pipelines.

## The known weakness: free-form tags are not governable

Oracle's own tagging documentation is direct about this:

> "To create metadata that you can trust to manage resources and collect data,
> use defined tags."

This repository uses **free-form** tags, and free-form tags have real limits:

| | Free-form | Defined |
| --- | --- | --- |
| IAM policy can control who applies or changes them | No | Yes |
| Tag defaults can apply them automatically | No | Yes |
| Values validated against a predefined list | No | Yes |
| Usable for cost tracking | No | Yes |

In other words, the ownership contract this repository depends on is currently
written in the one kind of metadata OCI cannot protect. Anyone with write
permission on a database can edit `exacs_resource_key` in the console. If they
clear it, the resource becomes invisible to the automation; if they duplicate
it onto a second resource, the `length == 1` assertion fires and the operation
refuses to run. The second case is loud and safe. The first is silent.

### Why free-form tags anyway, for now

Defined tags require a tag namespace to exist before the first resource can be
created. That is a one-time tenancy-level prerequisite, and it needs a
different set of IAM permissions than the operations themselves. For an MVP
whose stated goal is that a customer new to OCI and Ansible can run the first
operation without preparing anything, that prerequisite is a real cost.

Free-form tags need no setup and behave identically for resolution purposes.
The difference is entirely about governance, not about whether the automation
works.

### What to do about it

Two mitigations, in order of effort:

1. **Protect the tags with IAM.** Grant the automation principal permission to
   manage databases, and do not grant broad `use` permissions on those
   resources to humans in the same compartment. This does not make free-form
   tags governable, but it reduces who can quietly change one.
2. **Move to defined tags.** Create a tag namespace, for example `exacs`, with
   the keys `project`, `environment`, and `resource_key`, and switch the
   filters from `freeform_tags.exacs_project` to
   `defined_tags.exacs.project`. This is the recommended end state for a
   governed environment, and the change is mechanical: the resolution logic and
   the safety invariant are unchanged, only the tag location moves.

The second is the documented path for customers who already run an OCI landing
zone, since those tenancies normally have a governed tag namespace already.

## Consequences a customer should know about

- **Pre-existing resources are invisible.** A database created before this
  automation, or by hand, has no ownership tags, so no operation here will
  find it or touch it. That is deliberate: the automation only manages what it
  owns. Adopting an existing database means tagging it first.
- **The tags must never be removed.** They are not decoration. Removing them
  orphans the resource from the automation's point of view.
- **Resolution costs API calls.** Each run lists Database Homes and then the
  databases inside each one, because OCI refuses to list databases by
  compartment alone. This is linear in the number of Database Homes and is not
  a concern at the scale of a single VM Cluster.

## Sources

- [OCI Tagging Overview](https://docs.oracle.com/en-us/iaas/Content/Tagging/Concepts/taggingoverview.htm)
- [OCI Tags and Tag Namespace Concepts](https://docs.oracle.com/en-us/iaas/Content/Tagging/Tasks/managingtagsandtagnamespaces.htm)
- [OCI Tagging Enhances Cloud Governance](https://blogs.oracle.com/database/oci-tagging-enhances-cloud-governance)
- [HashiCorp Well-Architected Framework: create and implement a cloud resource tagging strategy](https://developer.hashicorp.com/well-architected-framework/optimize-systems/lifecycle-management/tag-cloud-resources)
- [AWS Best Practices for Tagging: automated infrastructure activities](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/automated-infrastructure-activities.html)
