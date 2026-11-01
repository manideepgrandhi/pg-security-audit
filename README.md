# pg-security-audit

Quarterly permissions audit for PostgreSQL and Aurora PostgreSQL fleets. **Status: planning — no code yet.**

## Goal
Automate the quarterly security audit and save the team manual hours: track and store all permission information on each server, query it on the fly from a jump box or laptop, and generate HTML + CSV reports of users with elevated roles and elevated permissions (write, create, alter, drop, ownership).

## Design (no central database)
Each server keeps its own audit data in a `secaudit` schema (in `postgres` or a chosen DBA database). A Bash/psql runner on a jump box or laptop calls the functions remotely using a common audit user and builds the reports.

### Tables (per server, stamped with run id + timestamp)
| Table | Contents |
|---|---|
| `runs` | Run id, quarter label, start/end, status |
| `roles` / `role_members` | Role attributes (superuser, createrole, createdb, replication, bypassrls, login, valid until) and expanded membership chains (`rds_superuser`, `pg_write_all_data`, other `pg_*` roles) |
| `db_privs` | Database level: CONNECT, CREATE, TEMP, owner |
| `schema_privs` | Schema level: USAGE, CREATE, owner |
| `object_privs` | Object level: SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, EXECUTE on tables, views, sequences, functions; owners |
| `default_privs` | `ALTER DEFAULT PRIVILEGES` by grantor, schema, object type, grantee |

Each grant notes whether it is direct, inherited through a role, or via PUBLIC.

### Functions
- `secaudit.snapshot()` – role and database-level capture; `secaudit.snapshot_db()` – schema/object/default privileges, run in each database.
- `secaudit.elevated_users()` – superusers, rds_superuser members, createrole/createdb/replication.
- `secaudit.write_ddl_access()` – write and DDL rights (INSERT/UPDATE/DELETE/TRUNCATE, CREATE, ownership => ALTER/DROP).
- `secaudit.public_grants()` – anything granted to PUBLIC.
- `secaudit.diff(prev_run, this_run)` – quarter-over-quarter changes.

### Runner: `pg_security_audit.sh`
- Reads a server list (host, port, label); auth via `~/.pgpass` (never passwords on the command line).
- Per server: run snapshots, pull report rows as CSV.
- Outputs per-server CSVs plus one fleet HTML page (tabs: elevated users, write/DDL access, default privileges, PUBLIC grants, changes since last quarter; filters + CSV export; offline).
- `--merge` to combine saved runs; `--install` to deploy/upgrade the schema (separate from audit runs); `--no-store` for zero writes on servers.
- Unreachable servers are reported as errors without stopping the run.

## Open questions
1. Who runs `--install` (common user vs. an `rds_superuser` once)?
2. Hide system schemas and AWS internal roles (`rdsadmin`, `rds_*`) by default?

## License
MIT
