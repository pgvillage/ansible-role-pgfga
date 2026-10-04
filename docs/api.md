# pgvillage.pgfga – Role API

This document describes all variables that can be used to configure the `pgvillage.pgfga` role.
Defaults are defined in [`defaults/main.yml`](../defaults/main.yml).

## Overview

| Variable | Default | Description |
| --- | --- | --- |
| `pgfga_deploydir` | `/usr/local/bin` | Directory for the pgfga binary |
| `pgfga_configdir` | `/etc/pgfga` | Directory for the pgfga configuration file |
| `pgfga_osuser` | `pgfga` | OS user that runs pgfga |
| `pgfga_osgroup` | `pgfga` | Primary group of `pgfga_osuser` |
| `pgfga_package_state` | `present` | State of the pgfga packages |
| `pgfga_packages` | `['pgfga']` | Packages to install from repositories |
| `pgfga_local_packages` | `[]` | Local package files to copy and install |
| `pgfga_cert_managed` | `false` | Deploy client certificates for pgfga |
| `pgfga_config` | `{}` | pgfga configuration (rendered to `config.yaml`) |

## Directories

### `pgfga_deploydir`

- **Type:** string
- **Default:** `/usr/local/bin`

Directory that is created to hold the pgfga binary.

### `pgfga_configdir`

- **Type:** string
- **Default:** `/etc/pgfga`

Directory that holds the pgfga configuration file. The role renders `pgfga_config` into
`{{ pgfga_configdir }}/config.yaml` (owner `root:root`, mode `0644`).

## OS user and group

### `pgfga_osuser`

- **Type:** string
- **Default:** `pgfga`

OS user that pgfga runs as. It is created as a system user with a home directory.
A `~/.postgresql` directory (mode `0750`) is created in its home directory, which is where client
certificates are deployed when `pgfga_cert_managed` is enabled.

### `pgfga_osgroup`

- **Type:** string
- **Default:** `pgfga`

Primary OS group of `pgfga_osuser`. It is created as a system group.

## Packages

### `pgfga_package_state`

- **Type:** string
- **Default:** `present`

State of the pgfga packages, passed to `ansible.builtin.package` (e.g. `present`, `latest`, `absent`).
Applies to both `pgfga_packages` and `pgfga_local_packages`.

### `pgfga_packages`

- **Type:** list of strings
- **Default:** `['pgfga']`

List of packages to install from the package repositories configured on the target host.

### `pgfga_local_packages`

- **Type:** list of strings
- **Default:** `[]`

List of local package files (relative to the role's `files` directory or absolute paths on the controller).
Each file is copied to `/tmp` on the target and installed from there. Useful for air-gapped environments.

```yaml
pgfga_local_packages:
  - pgfga-1.0.0-1.x86_64.rpm
```

## Certificates

### `pgfga_cert_managed`

- **Type:** boolean
- **Default:** `false`

When `true`, client certificates are deployed to `~/.postgresql` of `pgfga_osuser`, so pgfga can
authenticate to PostgreSQL using certificates:

| File | Source variable | Mode |
| --- | --- | --- |
| `root.crt` | `certs.client.postgres` | `0640` |
| `postgresql.crt` | `certs.client.pgfga` | `0640` |
| `postgresql.key` | `private_keys.client.pgfga` | `0600` |

These source variables are not defined by this role and must be provided elsewhere (e.g. in `group_vars`).

```yaml
pgfga_cert_managed: true
certs:
  client:
    postgres: "{{ lookup('file', 'root.crt') }}"
    pgfga: "{{ lookup('file', 'pgfga.crt') }}"
private_keys:
  client:
    pgfga: "{{ lookup('file', 'pgfga.key') }}"
```

## Configuration

### `pgfga_config`

- **Type:** dictionary
- **Default:** `{}`

The pgfga configuration. It is rendered as YAML (via `to_nice_yaml`) into
`{{ pgfga_configdir }}/config.yaml`. See the pgfga documentation for the supported structure.
After the configuration is deployed, the `pgfga` systemd service is (re)started and enabled.
