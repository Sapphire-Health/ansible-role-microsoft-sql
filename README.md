# microsoft_sql

Ansible role for installing and configuring Microsoft SQL Server on Windows. The role installs SQL
Server (and optionally SQL Server Management Studio), opens the firewall, and configures logins,
databases, and database permissions.

## Requirements

- Windows host(s) reachable over WinRM/PSRP.
- The following Ansible collections on the control node:
  - `ansible.windows`
  - `community.windows`
- Minimum Ansible version `2.10`.
- Storage provisioned on the target host (the role checks for at least two partitions by default).
- A SQL Server installation source (ISO or extractable installer), delivered via HTTP(S) or uploaded
  from the control node.

The following PowerShell modules are installed on the host by the `install_dependencies` stage:
`SQLServerDsc`, `SqlServer`, `StorageDsc`, `ServerManager`, `dbatools`, `xNetworking`.

## Role Variables

### Required

| Variable                              | Description                                                                                  |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| `microsoft_sql_install_source_file`   | Name of the SQL install file (e.g. `SQLServer2022-x64-ENU-Dev.iso`). ISOs are mounted; other files are extracted. |
| `microsoft_sql_sa_password`           | Password for the SQL `sa` account. May instead be supplied via the `MICROSOFT_SQL_SA_PASSWORD` environment variable. |
| `microsoft_sql_dsc_sqlsysadminaccounts` | List of accounts granted the SQL sysadmin role. SQL setup requires at least one.            |

### Optional (defaults)

| Variable                                | Default                          | Description                                                            |
| --------------------------------------- | -------------------------------- | ---------------------------------------------------------------------- |
| `microsoft_sql_install_source_dir`      | `{{ shared_files_directory }}`   | Directory or URL the SQL media is retrieved from. A URL is downloaded on the host; a path is uploaded from the control node. |
| `microsoft_sql_temp_dir`                | `C:/Windows/Temp/sql_install`    | Working directory on the host for install media.                       |
| `microsoft_sql_force_download`          | `false`                          | Re-download the media even if it already exists in the temp directory. |
| `microsoft_sql_delete_temp_dir`         | `true`                           | Delete the temp directory after the install/SSMS stages complete.      |
| `microsoft_sql_storage_check`           | `true`                           | Require at least two partitions before installing. Set `false` to skip. |
| `microsoft_sql_install_psmodules`       | see [Requirements](#requirements) | PowerShell modules installed during `install_dependencies`.           |
| `microsoft_sql_install_smss`            | `true`                           | Install SQL Server Management Studio. Set `false` to skip.             |
| `microsoft_sql_smss_install_source_dir` | `https://aka.ms/ssms/22/release` | Directory or URL the SSMS installer is retrieved from.                 |
| `microsoft_sql_smss_install_source_file`| `vs_SSMS.exe`                    | Name of the SSMS installer file.                                       |
| `microsoft_sql_dsc_resource_name`       | `SQLSetup`                       | Name of the DSC resource used to install SQL Server.                   |
| `microsoft_sql_dsc_action`              | `Install`                        | DSC setup action.                                                      |
| `microsoft_sql_dsc_instancename`        | `MSSQLSERVER`                    | SQL Server instance name.                                              |
| `microsoft_sql_dsc_features`            | `SQLENGINE`                      | Comma-separated SQL Server features to install.                        |
| `microsoft_sql_dsc_securitymode`        | `SQL`                            | Authentication mode for the instance.                                  |
| `microsoft_sql_dsc_installsqldatadir`   | _(omitted)_                      | Root data directory for the instance.                                  |
| `microsoft_sql_dsc_sqluserdbdir`        | _(omitted)_                      | Default directory for user database data files.                        |
| `microsoft_sql_dsc_sqluserdblogdir`     | _(omitted)_                      | Default directory for user database log files.                         |
| `microsoft_sql_dsc_sqltempdbdir`        | `T:\TempDB`                      | Directory for tempdb data files.                                       |
| `microsoft_sql_dsc_sqltempdblogdir`     | `T:\TempDB`                      | Directory for tempdb log files.                                        |
| `microsoft_sql_dsc_sqlbackupdir`        | _(omitted)_                      | Default backup directory.                                              |
| `microsoft_sql_license_key`             | _(omitted)_                      | Product key for licensed editions. May instead be supplied via the `MICROSOFT_SQL_LICENSE_KEY` environment variable. |
| `microsoft_sql_databases`               | `[]`                             | Databases to create and configure. See [Databases](#databases).        |

### Passwords

Sensitive values may be supplied as an Ansible variable or an environment variable. When both are
set, the **Ansible variable takes precedence** and the environment variable is used as a fallback.
Store secrets in an Ansible Vault rather than committing them.

| Ansible Variable                | Environment Variable            | Description                                  |
| ------------------------------- | ------------------------------- | -------------------------------------------- |
| `microsoft_sql_sa_password`     | `MICROSOFT_SQL_SA_PASSWORD`     | Password for the SQL `sa` account.           |
| `microsoft_sql_<user>_password` | `MICROSOFT_SQL_<USER>_PASSWORD` | Password for each local SQL login.           |
| `microsoft_sql_license_key`     | `MICROSOFT_SQL_LICENSE_KEY`     | Product key for licensed editions.           |

For local SQL logins the resolution order is: a `password` attribute on the user object, then the
`microsoft_sql_<user>_password` Ansible variable, then the `MICROSOFT_SQL_<USER>_PASSWORD`
environment variable.

## Databases

`microsoft_sql_databases` is a list of database objects:

| Key                   | Required | Description                                                         |
| --------------------- | -------- | ------------------------------------------------------------------- |
| `name`                | yes      | Database name.                                                      |
| `collation`           | no       | Database collation.                                                 |
| `compatibility_level` | no       | Database compatibility level.                                       |
| `recovery_model`      | no       | Recovery model (defaults to `FULL`).                                |
| `owner_name`          | no       | Database owner.                                                     |
| `db_size` / `db_max_size` / `db_growth`    | no | Data file size, max size, and growth. Required to configure data files.  |
| `log_size` / `log_max_size` / `log_growth` | no | Log file size, max size, and growth. Required to configure log files.    |
| `users`               | no       | List of login names to add to the database.                         |
| `roles`               | no       | List of `{ name, members }` objects assigning database roles.       |

## Tags

| Tag                    | Stage                                                            |
| ---------------------- | ---------------------------------------------------------------- |
| `install_dependencies` | Install PowerShell modules and Windows features.                 |
| `install_sql`          | Download/mount the media and install SQL Server.                 |
| `configure_firewall`   | Open the firewall for SQL Server (TCP 1433-1434).                |
| `install_ssms`         | Install SQL Server Management Studio.                            |
| `sql_users`            | Create SQL logins (local SQL and domain users).                  |
| `sql_databases`        | Create databases and configure data/log files.                  |
| `sql_permissions`      | Add users to databases and assign database roles.                |

Use `--tags` to run only specific stages or `--skip-tags` to exclude them.

## Example Playbook

```yaml
- hosts: sql_servers
  roles:
    - role: microsoft_sql
      vars:
        microsoft_sql_install_source_file: SQLServer2022-x64-ENU-Dev.iso
        microsoft_sql_dsc_sqlsysadminaccounts:
          - sapphire\epicadmin
        microsoft_sql_databases:
          - name: AppDb
            db_size: 4096 MB
            db_max_size: 10240 MB
            db_growth: 256 MB
            log_size: 1024 MB
            log_max_size: 10240 MB
            log_growth: 500 MB
            users:
              - appuser
            roles:
              - name: db_owner
                members:
                  - appuser
```

## License

MIT

## Author Information

Lyas Spiehler — Sapphire Health
