# Getting started from a clean runner

## Required components

Install these components on the runner that will execute the master playbooks:

- Python 3.12 and its virtual-environment package.
- Git, to obtain this repository.
- Ansible `14.2.0` (`ansible-core` 2.21).
- The OCI Python SDK (`oci` package).
- OCI Ansible Collection `oracle.oci` `5.5.0`.

The OCI CLI is **not required** for the API-based masters currently included.
The collection calls OCI through the Python SDK. It is nevertheless the
approved fallback when OCI exposes an operation in the CLI before the Ansible
collection provides a suitable module. Install it when that fallback,
interactive diagnosis, or `oci setup config` is needed.

## Install Ansible, SDK, and collection

On a Linux runner, install Python 3.12 using the platform package manager, then
create an isolated environment in the cloned repository:

```bash
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible==14.2.0 oci==2.183.0
ansible-galaxy collection install -r ansible/requirements.yml
ansible --version
ansible-galaxy collection list oracle.oci
```

The pinned collection version is held in `ansible/requirements.yml`. Do not use
the operating-system Python for automation dependencies: the virtual
environment makes the installed versions reproducible.

## Optional OCI CLI

Install the CLI in the same virtual environment when it is needed for an
operator workflow or a controlled collection-gap fallback:

```bash
python -m pip install oci-cli
oci --version
```

For API-key authentication, an operator can create a protected local SDK/CLI
profile with `oci setup config`, then point the collection at that profile with
`OCI_CONFIG_FILE` and `OCI_CONFIG_PROFILE`. Keep the resulting PEM and config
file outside Git. For Instance Principal, neither OCI CLI nor an SDK config
file is required.

## Verify before execution

Run a playbook syntax check with a fictional example before supplying real
values or credentials:

```bash
ansible-playbook --syntax-check ansible/playbooks/exacs-db-home-create.yml \
  -e operation_file=$PWD/ansible/example/yaml/db-home-create.example.yml
```

This check does not contact OCI, so it does not prove that an operation
succeeds against a tenancy. Only a `precheck` run against your own tenancy does
that, and [day2-operations.md](day2-operations.md) records how far each
operation has been proven. Maintainers can also run `ansible-lint ansible/`.

Then follow [authentication.md](authentication.md) and the
[customer deployment guide](customer-guide.md). A `precheck` run is the first
OCI call for every mutating operation.
