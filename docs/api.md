# pgvillage.pgfga – Role API

This document describes all variables that can be used to configure the `pgvillage.pgfga` role.
Defaults are defined in [`defaults/main.yml`](../defaults/main.yml).

## Overview

| Variable | Default | Description |
| --- | --- | --- |
| `pgfga_deploydir` | `/usr/bin` | Directory for the pgfga binary (used in the systemd unit) |
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
- **Default:** `/usr/bin`

Directory that is created to hold the pgfga binary.
The systemd unit runs `<pgfga_deploydir>/pgfga`.

When the role installs packages (state not `absent`) and `<pgfga_deploydir>/pgfga` does not exist
afterwards, the role fails before it changes the unit and config. Older pgvillage packages installed
pgfga in `/usr/local/bin`: set `pgfga_package_state: latest` once to move to the new package, or set
`pgfga_deploydir: /usr/local/bin` to keep the old build.

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

## PostgreSQL login

pgfga runs as `pgfga_osuser` and connects to PostgreSQL over the unix socket as `pgfga_pg_user`, with ident (peer)
authentication: no password is needed. The PostgreSQL user has the same name as the OS user, so a
`local all all ident` (or `peer`) entry in `pg_hba.conf` lets it in. The role creates that user (`LOGIN SUPERUSER`)
with `community.postgresql.postgresql_user`, as `pgfga_pg_admin_osuser` over the socket; on a standby it creates
nothing (the user comes from the primary). The module needs psycopg on the host (`pgfga_python_packages`).

| Variable | Default | Description |
|----------|---------|-------------|
| `pgfga_pg_user` | `{{ pgfga_osuser }}` | PostgreSQL user pgfga logs in as. |
| `pgfga_pg_host` | `/var/run/postgresql` | Unix socket directory of PostgreSQL (e.g. `/tmp` for stolon in PgVillage). |
| `pgfga_pg_port` | `5432` | Port of PostgreSQL. |
| `pgfga_pg_dbname` | `postgres` | Database pgfga connects to. |
| `pgfga_pg_create_user` | `true` | Create `pgfga_pg_user` when it doesn't exist. |
| `pgfga_pg_admin_osuser` | `postgres` | OS user that creates `pgfga_pg_user` (a PostgreSQL superuser over peer authentication). |
| `pgfga_python_packages` | `python3-psycopg2` (`python313-psycopg2` on zypper) | psycopg for the `community.postgresql` modules, installed when `pgfga_pg_create_user` is true. Built from `_pgfga_python_packages` in [vars/main.yml](../vars/main.yml) (internal; an inventory can't override it, set `pgfga_python_packages` instead). |
| `pgfga_default_config` | `postgresql_dsn` from the variables above | Default pgfga configuration; `pgfga_config` is merged over it. |

## Configuration

### `pgfga_config`

- **Type:** dictionary
- **Default:** `{}`

The pgfga configuration (databases, roles, users, ...). It is merged recursively over `pgfga_default_config` (the
connection, see [PostgreSQL login](#postgresql-login)) and rendered as YAML (via `to_nice_yaml`) into
`{{ pgfga_configdir }}/config.yaml`. See the pgfga documentation for the supported structure.
After the configuration is deployed, the `pgfga` systemd service is (re)started and enabled.
