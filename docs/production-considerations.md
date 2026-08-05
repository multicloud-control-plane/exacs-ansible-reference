# Moving from reference to production

This repository is deliberately a single, self-contained reference: one repo,
no external dependency beyond the pinned `oracle.oci` collection, so a
customer new to Ansible can clone it and run the first operation with nothing
else to set up. Two specific trade-offs support that goal, and both have a
documented path to a more governed state once this stops being a learning
exercise and starts being an operated platform. Free-form tags versus defined
tags is one; see [resource-identity.md](resource-identity.md). This document
covers the other.

## Separation of duties: split the operation catalog from its implementation

Today, `ansible/playbooks/common/exacs/` (the mutating logic — resolvers, API
calls, rescue blocks) lives in the same repository and the same review
process as `ansible/playbooks/exacs-*.yml` (the thin masters that decide
which operations exist at all). One reviewer approving a change here is
implicitly approving both "is this operation allowed to exist" and "does
this specific API call do the right thing." For a single learner repository
that is fine. It stops being fine once multiple teams depend on the same
automation and no single reviewer should hold both keys.

### The recommended split

Package `common/exacs/` as its own versioned Ansible Collection — the same
mechanism this repository already depends on for `oracle.oci`:

- **A "common" repository**: the Collection. Each operation becomes a role
  (`roles/cdb_create/tasks/{precheck,apply,verify}.yml`), plus the shared
  resolvers. Reviewed and released by whoever owns the mutating logic.
- **A "masters" repository**: the operation catalog. Each master pins a
  Collection version in its own `requirements.yml` and imports tasks from it
  by role name, exactly as it imports `oracle.oci` modules today:

  ```yaml
  # requirements.yml
  collections:
    - name: your_org.exacs_operations
      version: "1.2.0"
  ```

  ```yaml
  # exacs-cdb-create.yml
  tasks:
    - name: Precheck the CDB creation
      ansible.builtin.import_role:
        name: your_org.exacs_operations.cdb_create
        tasks_from: precheck
  ```

Changing how an operation behaves now requires a released, versioned change
to the Collection before the masters repository can adopt it — a deliberate
extra gate, not an accident of tooling.

### Why this reference does not do that already

The same reason it does not require a tag namespace up front: it is a cost
this MVP's stated audience should not have to pay to run the first
operation. A customer new to the OCI Ansible Collection would also need to
understand what a Collection is, how to host or reference one, and how two
repositories' version numbers relate to each other — three new concepts
before anything runs. It also breaks the workshop's teaching mechanism
directly: building a new operation means copying the closest existing one
and adapting it *inside the same repository*. Once the implementation lives
in a versioned external dependency, that exercise becomes forking or
patching a dependency instead.

### When to make the split

When an internal platform team starts operating this for consumers who did
not write it — the point where "review everything in one pull request" stops
being an adequate control, and separating who can change *behavior* from who
can change *what is offered* becomes a real requirement rather than a
hypothetical one.
