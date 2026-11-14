
# clickhouse-backup (version 1.2.0)

This tool, clickhouse-backup, saves backups of all databases located on the current ClickHouse server
to the `/srv/clickhouse-backup/YYYY-MM-DD-HHMMSS-clickhouse-backup` directory:
one `.tar.zst` archive per database, created by the native `BACKUP` statement of ClickHouse.

Every backup directory is complete and independent of all other backup directories.

> [!WARNING]
> Data of tables with the `Memory`, `File`, `Set`, `Join` and `Buffer` engines is **NOT** saved to the backup,
> only the definitions of such tables are saved. See [What is saved, and what is not](#what-is-saved-and-what-is-not).

```
/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/
    2026-11-14-000001--globals-only.tar.zst    users, roles, quotas, profiles, row policies, functions, named collections
    2026-11-14-000001--config-only.tar.zst     /etc/clickhouse-server and /etc/clickhouse-client
    2026-11-14-000001--schema-only.tar.zst     definitions of all databases and tables, without data
    2026-11-14-000002-default.tar.zst          one archive per database: definitions and data
    2026-11-14-000002-mydatabase.tar.zst
    CHECKSUM                                   SHA256 of every archive
```

Tested with ClickHouse 26.10 on Rocky Linux 10. The code is compatible with Python 3.9 (Rocky Linux 9).

## Requirements

- single node ClickHouse server, clickhouse-backup must be started on the same host as root
- `clickhouse-client`, `zstd`, `tar`
- Python 3.9 or newer with the `systemd` module, and the `tomli` module if Python is older than 3.11
- free space in the temporary directory (`/var/tmp` by default) equal to the uncompressed size of the largest database

```
dnf install zstd tar python3-systemd
python3 -c 'import tomllib' || python3 -m pip install tomli
```

## Installation

```
cd /opt ; git clone https://github.com/makhomed/clickhouse-backup.git
```

## Upgrade

```
cd /opt/clickhouse-backup ; git pull
```

## Usage

```
/opt/clickhouse-backup/clickhouse-backup
```

Options:

```
--config FILE              configuration file (default: /opt/clickhouse-backup/clickhouse-backup.toml)
--compression-level LEVEL  zstd compression level from 1 to 19, overrides every level of the configuration file
--version                  print the version
```

Exit code is `0` only if a complete backup was created. Errors and warnings are printed to stderr,
the full log including statistics is written to the journal: `journalctl -t clickhouse-backup`.

## Automation

```
# cat /etc/cron.d/clickhouse-backup
```
```
0 0 * * * root /opt/clickhouse-backup/clickhouse-backup
```

## Configuration

The configuration file `/opt/clickhouse-backup/clickhouse-backup.toml` is optional.
All keys with their default values are described in `clickhouse-backup.toml.example`.

```
backup_save_days = 14
compression_level = 3
exclude_databases = ["scratch"]
exclude_tables.default = ["debug_log", "import_buffer"]

# settings of one database
databases.mydatabase.compression_level = 9
databases.mydatabase.compression_long = true
databases.mydatabase.warning_threshold_hours = 12
```

The compression level is taken from the command line, then from the settings of the database,
then from the global `compression_level`, then the built-in default `3` is used:
it is the default level of zstd itself.

Three settings can be set for one database: `compression_level`, `compression_long`
and `warning_threshold_hours`.

### Compression threads and memory

`compression_threads` is the number of threads of one `zstd` process, the default is `4`.
More threads compress faster only while the disks keep up, and they take CPU cores and memory
away from the ClickHouse server. Memory of one `zstd` process, measured with zstd 1.5.5:

| level | 1 thread | 4 threads | 8 threads |
|------:|---------:|----------:|----------:|
|     3 |   50 MiB |    90 MiB |   150 MiB |
|     9 |          |   200 MiB |   340 MiB |
|    19 |  190 MiB |   590 MiB |  1100 MiB |

`compression_threads = "auto"` uses half of the CPU cores: on a server with 96 cores it is 48 threads,
which is about 1 GiB at level 3 and several GiB at level 19. With `jobs` greater than 1
several `zstd` processes may run at the same time.

### Server configuration

ClickHouse reads and writes backups only in the directories listed in `backups.allowed_path`.
If `backup_root` or the temporary directory `/var/tmp/clickhouse-backup` is not in this list, clickhouse-backup
writes `/etc/clickhouse-server/config.d/clickhouse-backup.xml` which adds them, all other allowed paths are kept.
This normally happens once, on the first run. ClickHouse applies this change without a restart.
clickhouse-backup never restarts the server: if the change is not applied within 30 seconds,
it exits with an error and asks to execute `SYSTEM RELOAD CONFIG` or to restart the server.

## How it works

For each database:

1. `BACKUP DATABASE db TO File('<temporary directory>/<name>.tar') ... ASYNC` is started
   and polled through `system.backups`.
2. The size of `<name>.tar` is compared with the size reported by the server.
3. `<name>.tar` is compressed by `zstd` to `<name>.tar.zst` in the backup directory, then `<name>.tar` is removed.
   ClickHouse can write `.tar.zst` itself, but it does not allow to choose the compression level for tar archives.
4. The archive is verified: `zstd --test`, then `tar --list` must find the backup manifest `.backup` in it.

The archives are written to the directory `<timestamp>-clickhouse-backup.partial`.
It is renamed to `<timestamp>-clickhouse-backup` only when all archives are created and verified,
so a directory without a suffix is always a complete backup.

### Temporary files

Uncompressed tar archives are written to `<temporary_directory>/clickhouse-backup/<timestamp>-<random>/`,
`temporary_directory` is `/var/tmp` by default. Free space equal to the uncompressed size of the largest
database is required there.

`/var/tmp` is world-writable, so the temporary files are protected from other local users:

- `/var/tmp/clickhouse-backup` must be a real directory owned by the `clickhouse` user with mode `0700`.
  It is created with `mkdir`, which fails if a file or a symbolic link with this name already exists.
  If something else is found there, clickhouse-backup refuses to work.
- Each run creates a subdirectory with an unpredictable name inside it and removes it at the end.
- The directory with a fixed name is required because ClickHouse accepts only paths listed in `backups.allowed_path`.

#### PrivateTmp

clickhouse-backup works both with `PrivateTmp=no` and with `PrivateTmp=yes` of `clickhouse-server.service`,
no configuration is required.

With `PrivateTmp=yes` the server has its own `/tmp` and `/var/tmp`, invisible to other processes.
`temporary_directory` is always a path inside the file system of the server process. clickhouse-backup reads
the PID of the server from the `status` file in the data directory of the server, compares the mount namespaces
of both processes, and if they differ, it works with the temporary files through `/proc/<pid>/root`.
The journal then contains the line `clickhouse-server has its own mount namespace`.

With `PrivateTmp=yes` systemd also mounts everything listed for `/tmp` and `/var/tmp` in `/etc/fstab` before
the service starts. Check where `/var/tmp` of the server really is, and how much space is there:

```
df -h /proc/$(systemctl show clickhouse-server -p MainPID --value)/root/var/tmp
```

### Failures

- If the backup of at least one database fails, the directory is renamed to `<timestamp>-clickhouse-backup.failed`.
  It contains only the verified archives of other databases. The exit code is `1`.
- Complete backups older than `backup_save_days` are removed only after a successful run, so repeated
  failures never rotate the last good backups away. Failed backups are removed by age after any run.
- On SIGTERM, SIGINT, SIGQUIT or SIGHUP the running `BACKUP` query is killed on the server,
  the temporary files and the `.partial` directory are removed.
- If clickhouse-backup was killed with SIGKILL or the host was rebooted, the next run kills the unfinished
  backups left on the server and removes the temporary files and the `.partial` directories.
- Only one instance can run at a time (`/run/clickhouse-backup.lock`).

## What is saved, and what is not

- Databases `system`, `INFORMATION_SCHEMA` and `information_schema` are never saved.
  Server logs (`system.query_log` and so on) are not saved.
- Users, roles and other access entities created by SQL are saved to the `globals-only` archive.
  Users defined in `users.xml` and `users.d` are saved to the `config-only` archive.
- Data is saved for the engines: `*MergeTree`, `Log`, `TinyLog`, `StripeLog`, `MaterializedView`,
  `EmbeddedRocksDB`, `KeeperMap`, `TimeSeries`.
- **Data of `Memory` tables is NOT saved, only their definitions.** After a restore such tables are empty.
  This is intentional: ClickHouse 26.10 is unable to restore data of the `Memory` engine from a tar archive,
  `RESTORE` fails with `getPosition not supported when reading from archive` and leaves the database
  partially restored. Data of this engine is lost on every restart of the server anyway.
- **Data of `File`, `Set`, `Join` and `Buffer` tables is NOT saved, only their definitions**:
  the `BACKUP` statement of ClickHouse does not support data of these engines.
- For all other engines (`View`, `Dictionary`, `Distributed`, `Kafka`, `MySQL`, `PostgreSQL` and so on)
  only the definition is saved too: they keep no data of their own. A dictionary is loaded from its source
  again after the restore, so the source must be available.
- clickhouse-backup prints a warning for every table whose local data is not saved.
  Add the table to `exclude_tables` to silence the warning, then its definition is not saved either.

### Consistency

- The set of databases and tables is a consistent snapshot.
- Every `MergeTree` table is saved as an atomic snapshot of its data parts.
  Unfinished mutations are saved and continue after the restore.
- There is no single point in time for different tables: if data is inserted during the backup,
  one table may be a moment older than another. ClickHouse has no transactions across tables,
  only a file system snapshot of a stopped or idle server gives a single point in time for all tables.

## Restore

All examples are for the backup directory:

```
cd /srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup
```

The archives must stay inside `backup_root`, or in any other directory listed in `backups.allowed_path`,
and must be readable by the `clickhouse` user. A database name with unusual characters is percent-encoded
in the file name, for example the database `my db` is saved to `...-my%20db.tar.zst`.

### 0. Verify the archives

```
sha256sum --check CHECKSUM
```

### 1. Restore a lost database

The database must not exist, or must be empty.

```
clickhouse-client --query "RESTORE DATABASE mydatabase FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000002-mydatabase.tar.zst')"
```

`RESTORE` refuses to write to a table which already contains data. To replace a damaged database, drop it first:

```
clickhouse-client --query "DROP DATABASE mydatabase SYNC"
```

### 2. Restore a database under another name

Useful to look into a backup or to copy lost rows back with `INSERT INTO ... SELECT`.

```
clickhouse-client --query "RESTORE DATABASE mydatabase AS mydatabase_restored FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000002-mydatabase.tar.zst')"
```

### 3. Restore one table

The target database must exist.

```
clickhouse-client --query "RESTORE TABLE mydatabase.events FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000002-mydatabase.tar.zst')"
clickhouse-client --query "RESTORE TABLE mydatabase.events AS mydatabase.events_restored FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000002-mydatabase.tar.zst')"
```

### 4. Restore users, roles and functions

```
clickhouse-client --query "RESTORE TABLE system.users, TABLE system.roles, TABLE system.settings_profiles, TABLE system.row_policies, TABLE system.quotas, TABLE system.functions FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000001--globals-only.tar.zst')"
```

`RESTORE` does not fail if these users, roles and functions already exist. To see what is inside the archive:

```
tar --zstd -xOf 2026-11-14-000001--globals-only.tar.zst --wildcards 'data/system/*'
```

### 5. Restore only the structure

```
clickhouse-client --query "RESTORE DATABASE mydatabase AS mydatabase_empty FROM File('/srv/clickhouse-backup/2026-11-14-000001-clickhouse-backup/2026-11-14-000001--schema-only.tar.zst') SETTINGS structure_only = true"
```

To read the definitions without the server:

```
tar --zstd -xOf 2026-11-14-000001--schema-only.tar.zst metadata/mydatabase/events.sql
```

### 6. Restore the server configuration

The `config-only` archive is an ordinary tar archive with paths relative to `/`.
Extract it to a temporary directory and copy the required files by hand.

```
mkdir /root/restored-config
tar --zstd -xf 2026-11-14-000001--config-only.tar.zst -C /root/restored-config
```

### 7. Restore the whole server on a new host

1. Install the same version of ClickHouse, restore the configuration (6) and start the server.
2. Copy the backup directory to `/srv/clickhouse-backup`, run `chown -R clickhouse:clickhouse /srv/clickhouse-backup`.
3. Install clickhouse-backup and run it once: it adds `/srv/clickhouse-backup` to `backups.allowed_path`.
   Or create `/etc/clickhouse-server/config.d/clickhouse-backup.xml` by hand:
   ```
   <clickhouse>
       <backups>
           <allowed_path>backups</allowed_path>
           <allowed_path>/srv/clickhouse-backup/</allowed_path>
       </backups>
   </clickhouse>
   ```
4. Restore the access entities (4), then every database (1).

### 8. If RESTORE is unable to read the archive

A `.tar.zst` archive is a plain container. Extract it to a directory and restore from the directory,
the trailing slash is required:

```
mkdir /srv/clickhouse-backup/extracted
tar --zstd -xf 2026-11-14-000002-mydatabase.tar.zst -C /srv/clickhouse-backup/extracted
chown -R clickhouse:clickhouse /srv/clickhouse-backup/extracted
clickhouse-client --query "RESTORE DATABASE mydatabase FROM File('/srv/clickhouse-backup/extracted/')"
rm -rf /srv/clickhouse-backup/extracted
```

### Progress and errors of RESTORE

A long `RESTORE` may fail in `clickhouse-client` with a receive timeout while the server continues the work.
Start it with the `ASYNC` keyword at the end of the statement and watch the status:

```
clickhouse-client --query "SELECT id, name, status, error, start_time, end_time FROM system.backups ORDER BY start_time DESC LIMIT 5 FORMAT Vertical"
```

Test the restore procedure regularly: a backup which was never restored is only a hope.

## License

GNU General Public License, version 3.0 or later, see the `LICENSE` file.
