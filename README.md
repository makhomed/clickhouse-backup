
# clickhouse-backup (version 1.7.0)

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

Tested with ClickHouse 26.10 on Rocky Linux 9 (Python 3.9) and Rocky Linux 10 (Python 3.12).

## Contents

- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Installation](#installation)
- [Upgrade](#upgrade)
    - [After an upgrade of ClickHouse](#after-an-upgrade-of-clickhouse)
- [Usage](#usage)
    - [Check before the first run](#check-before-the-first-run)
    - [Connection to the server](#connection-to-the-server)
- [Automation](#automation)
    - [cron](#cron)
    - [systemd timer](#systemd-timer)
    - [Monitoring](#monitoring)
- [Configuration](#configuration)
    - [Reference of the configuration keys](#reference-of-the-configuration-keys)
    - [Compression threads and memory](#compression-threads-and-memory)
    - [Server configuration](#server-configuration)
    - [Limiting the load](#limiting-the-load)
- [How it works](#how-it-works)
    - [Durability](#durability)
    - [Temporary files](#temporary-files)
    - [Free space](#free-space)
    - [Failures](#failures)
- [What is saved, and what is not](#what-is-saved-and-what-is-not)
    - [Secrets in the backup](#secrets-in-the-backup)
    - [Consistency](#consistency)
- [Restore](#restore)
    - [0. Verify the archives](#0-verify-the-archives)
    - [1. Restore a lost database](#1-restore-a-lost-database)
    - [2. Restore a database under another name](#2-restore-a-database-under-another-name)
    - [3. Restore one table](#3-restore-one-table)
    - [4. Restore users, roles and functions](#4-restore-users-roles-and-functions)
    - [5. Restore only the structure](#5-restore-only-the-structure)
    - [6. Restore the server configuration](#6-restore-the-server-configuration)
    - [7. Restore the whole server on a new host](#7-restore-the-whole-server-on-a-new-host)
    - [8. If RESTORE is unable to read the archive](#8-if-restore-is-unable-to-read-the-archive)
    - [Progress and errors of RESTORE](#progress-and-errors-of-restore)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Quick start

```
dnf install zstd tar python3-systemd
python3 -c 'import tomllib' || python3 -m pip install tomli

cd /opt ; git clone https://github.com/makhomed/clickhouse-backup.git

/opt/clickhouse-backup/clickhouse-backup --check      # changes nothing, shows what will be done
/opt/clickhouse-backup/clickhouse-backup              # the first backup
echo $?                                               # 0 means a complete backup
ls -l /srv/clickhouse-backup/*/

echo '0 0 * * * root /opt/clickhouse-backup/clickhouse-backup' > /etc/cron.d/clickhouse-backup
```

Then:

- restore one database under another name to be sure that the backup is usable:
  [Restore, 2](#2-restore-a-database-under-another-name);
- let the monitoring system watch the age of `/var/lib/clickhouse-backup/last-successful-backup`:
  [Monitoring](#monitoring);
- read [What is saved, and what is not](#what-is-saved-and-what-is-not).

No configuration file is required. The first run creates `/srv/clickhouse-backup`,
`/var/tmp/clickhouse-backup` and `/etc/clickhouse-server/config.d/clickhouse-backup.xml`,
the server is not restarted.

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

### After an upgrade of ClickHouse

clickhouse-backup depends on the behaviour of one particular version of the server. These facts were verified
with ClickHouse 26.10 and may change in another version:

- the list of table engines whose data is saved by the `BACKUP` statement;
- `compression_level` is rejected for tar archives, this is why the archive is compressed by an external `zstd`;
- data of the `Memory` engine can not be restored from a tar archive, this is why it is excluded;
- a synchronous `BACKUP` sends nothing to the client until it is finished, this is why `ASYNC` is used;
- `backups.allowed_path` is applied without a restart, and configuration elements with different attributes
  are appended when the configuration files are merged.

After every upgrade of ClickHouse to a new major version:

1. run `clickhouse-backup --check`;
2. run one backup and check that its exit code is `0`;
3. restore one database under another name ([Restore, 2](#2-restore-a-database-under-another-name)),
   compare the number of rows of its largest table with the original, and drop the restored database.

Restore was tested only with the same version of ClickHouse which has created the archive.
Keep the old backups until a backup made by the new version is restored successfully.

## Usage

```
/opt/clickhouse-backup/clickhouse-backup
```

Options:

```
--config FILE              configuration file (default: /opt/clickhouse-backup/clickhouse-backup.toml)
--compression-level LEVEL  zstd compression level from 1 to 19, overrides every level of the configuration file
--check                    change nothing, print what a real run would do
--version                  print the version
```

Exit code is `0` only if a complete backup was created. Errors and warnings are printed to stderr,
the full log including statistics is written to the journal: `journalctl -t clickhouse-backup`.

### Check before the first run

```
/opt/clickhouse-backup/clickhouse-backup --check
```

`--check` changes nothing and starts no backup. It prints the version and the mount namespace of the server,
the directories which will be created, whether the server configuration will be changed, the databases
and tables which will be saved or excluded with their sizes, the required and the available free space,
and the old backups which will be removed. Exit code is `0` if nothing prevents the backup.

### Connection to the server

clickhouse-backup starts `clickhouse-client` without a user name and a password, so `clickhouse-client` must be
able to connect by itself: with its configuration file in the home directory of root
(`~/.config/clickhouse/config.xml` or `~/.clickhouse-client/config.xml`), with `/etc/clickhouse-client/config.xml`,
or with the file named in the `clickhouse_client_config` key. clickhouse-backup was tested with the `default`
user which has all privileges; it executes `BACKUP`, `KILL QUERY` and `SELECT` from the `system` database.

## Automation

Use one of the two ways, not both.

### cron

```
# cat /etc/cron.d/clickhouse-backup
```
```
0 0 * * * root /opt/clickhouse-backup/clickhouse-backup
```

cron sends everything printed to stderr by e-mail, if a mail transfer agent is installed.

### systemd timer

```
# cat /etc/systemd/system/clickhouse-backup.service
```
```
[Unit]
Description=Backup of ClickHouse databases
After=clickhouse-server.service

[Service]
Type=oneshot
ExecStart=/opt/clickhouse-backup/clickhouse-backup
TimeoutStartSec=infinity

# optional limits for clickhouse-backup and for the zstd and tar processes started by it,
# the BACKUP statement itself is executed by the ClickHouse server and is not limited by them
#Nice=10
#CPUQuota=400%
#MemoryMax=4G
```
```
# cat /etc/systemd/system/clickhouse-backup.timer
```
```
[Unit]
Description=Daily backup of ClickHouse databases

[Timer]
OnCalendar=*-*-* 00:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
```
systemctl daemon-reload
systemctl enable --now clickhouse-backup.timer
```

- `systemctl list-timers clickhouse-backup.timer` shows the time of the last and of the next run.
- `systemctl status clickhouse-backup.service` shows the result of the last run,
  `journalctl -u clickhouse-backup.service -t clickhouse-backup` shows its log.
- `systemctl start clickhouse-backup.service` starts a backup now, `systemctl stop clickhouse-backup.service`
  cancels the running one cleanly.
- systemd never starts a second copy of a running service.
- `Persistent=true` starts a missed backup after the host was down at the scheduled time.
- With `MemoryMax` a `zstd` process which needs more memory is killed and the backup fails,
  see [Compression threads and memory](#compression-threads-and-memory).

### Monitoring

Neither cron nor systemd tells you that backups are not started at all. After each complete backup
clickhouse-backup rewrites the file `/var/lib/clickhouse-backup/last-successful-backup`:

```
2026-11-14T00:03:12Z 2026-11-14-000001-clickhouse-backup
```

Its modification time is the time of the last complete backup. Let the monitoring system raise an alarm
when the file is too old, for example older than 26 hours for a daily backup:

```
find /var/lib/clickhouse-backup/last-successful-backup -mmin -1560 | grep -q . || echo "NO FRESH BACKUP"
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

Unknown keys and values of a wrong type are errors: clickhouse-backup prints the name of the key and exits with code `1`.

### Reference of the configuration keys

| key | default | description |
|-----|---------|-------------|
| `backup_root` | `"/srv/clickhouse-backup"` | Root directory of the backups. Each run creates `<backup_root>/YYYY-MM-DD-HHMMSS-clickhouse-backup`, the time is in UTC. |
| `backup_save_days` | `7` | Complete backups older than this number of days are removed after a successful run. See [Failures](#failures). |
| `temporary_directory` | `"/var/tmp"` | Uncompressed archives are written to `<temporary_directory>/clickhouse-backup/<random name>`. See [Temporary files](#temporary-files). |
| `free_space_check` | `true` | Compare the size of the databases with the free space before the backup is started. See [Free space](#free-space). |
| `exclude_databases` | `[]` | Databases which are not saved. `system`, `INFORMATION_SCHEMA` and `information_schema` are never saved. |
| `exclude_tables.<database>` | none | List of tables of this database which are not saved: neither data nor definition. |
| `compression_level` | `3` | zstd compression level, from `1` to `19`. The option `--compression-level` overrides it. Can be set for one database. |
| `compression_long` | `false` | Long distance matching mode of zstd (`zstd --long`). Can be set for one database. |
| `compression_threads` | `4` | Threads of one `zstd` process: a number, `0` (all CPU cores) or `"auto"` (half of the CPU cores). See [Compression threads and memory](#compression-threads-and-memory). |
| `compression_nice` | `10` | Niceness of the `zstd` and `tar` processes, from `-20` to `19`, `0` means "do not change". |
| `max_backup_bandwidth` | `0` | Read speed limit of one `BACKUP` statement in bytes per second, `0` means unlimited. See [Limiting the load](#limiting-the-load). |
| `jobs` | `1` | Databases processed in parallel: a number or `"auto"` (half of the CPU cores). One `BACKUP` statement is already executed by many threads of the server. |
| `backup_timeout_hours` | `24` | The backup of one database is cancelled if it takes more than this number of hours. |
| `backup_status_retry_seconds` | `300` | How long failed status requests are repeated before a running backup is cancelled. See [Failures](#failures). |
| `warning_threshold_hours` | `4` | A warning is printed if the backup of one database takes more than this number of hours. Can be set for one database. |
| `databases.<database>.<key>` | none | Value of `compression_level`, `compression_long` or `warning_threshold_hours` for one database. |
| `success_file` | `"/var/lib/clickhouse-backup/last-successful-backup"` | File which is rewritten after each complete backup, an empty string disables it. See [Monitoring](#monitoring). |
| `config_directories` | `["/etc/clickhouse-server", "/etc/clickhouse-client"]` | Directories saved to the `config-only` archive. |
| `clickhouse_client_config` | `""` | Configuration file of `clickhouse-client` with the user name and the password. See [Connection to the server](#connection-to-the-server). |
| `server_config_file` | `"/etc/clickhouse-server/config.d/clickhouse-backup.xml"` | Server configuration file managed by clickhouse-backup. See [Server configuration](#server-configuration). |
| `server_config_reload_timeout_seconds` | `30` | How long to wait until the server applies a changed `server_config_file`. |
| `user`, `group` | `"clickhouse"` | Owner of the backup directories and of the archives: the user of the clickhouse-server process, because the server writes the archives. |
| `mode` | `0o700` | Mode of `backup_root` and of the backup directories. |
| `clickhouse_client` | `"/usr/bin/clickhouse-client"` | Path of the program. |
| `zstd` | `"/usr/bin/zstd"` | Path of the program. |
| `tar` | `"/usr/bin/tar"` | Path of the program. |
| `nice` | `"/usr/bin/nice"` | Path of the program. |

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
clickhouse-backup needs two of them: `backup_root` and the temporary directory `/var/tmp/clickhouse-backup`.
If they are not allowed, it writes `/etc/clickhouse-server/config.d/clickhouse-backup.xml`:

```
<!-- This file is managed by clickhouse-backup, do not edit it. -->
<clickhouse>
    <backups>
        <allowed_path clickhouse_backup="backup_root">/srv/clickhouse-backup/</allowed_path>
        <allowed_path clickhouse_backup="temporary_directory">/var/tmp/clickhouse-backup/</allowed_path>
    </backups>
</clickhouse>
```

- This normally happens once, on the first run. ClickHouse applies the change without a restart.
- clickhouse-backup never restarts the server: if the change is not applied within 30 seconds,
  it exits with an error and asks to execute `SYSTEM RELOAD CONFIG` or to restart the server.
- The `clickhouse_backup` attribute is essential. When ClickHouse merges configuration files, an element replaces
  the element with the same name and the same attributes, otherwise it is appended. An `allowed_path` element
  without attributes in `config.d` therefore replaces a path of the main configuration; the elements above
  are appended and never replace the paths allowed by the administrator.
- The file contains only what clickhouse-backup needs now. If `backup_root` or `temporary_directory` is changed,
  the old directory is removed from the file and stops being allowed.
- If the administrator has already allowed a directory or its parent in another configuration file,
  it is not repeated here. If both directories are allowed this way, the file is removed.
- clickhouse-backup refuses to overwrite a file with this name which it has not written itself.
- `clickhouse-backup --check` shows what will be changed.

### Limiting the load

A backup of one database has two phases, and they are limited in different ways.

| phase | who works | what is loaded | how to limit |
|-------|-----------|----------------|--------------|
| `BACKUP ... TO File('<name>.tar')` | the ClickHouse server | disk reads and writes | `max_backup_bandwidth` |
| `<name>.tar` to `<name>.tar.zst`, verification | `zstd` and `tar` started by clickhouse-backup | CPU, disk reads and writes | `compression_threads`, `compression_nice` |

- `max_backup_bandwidth` is the read speed limit of one `BACKUP` statement in bytes per second, `0` means unlimited.
  It is a query-level setting of ClickHouse, clickhouse-backup passes it with each `BACKUP` statement:
  the server configuration is not changed, no restart is required, other queries are not affected.
- `compression_nice` is the niceness of the `zstd` and `tar` processes, `10` by default.
- The number of threads which the server uses for backups is the server setting `backup_threads`, `16` by default.
  ClickHouse applies it only after a restart (`changeable_without_restart = No` in `system.server_settings`),
  so clickhouse-backup does not manage it. To change it, create a file in `/etc/clickhouse-server/config.d/`
  and restart the server:
  ```
  <clickhouse>
      <backup_threads>4</backup_threads>
  </clickhouse>
  ```
- For hard limits of CPU and memory of the whole clickhouse-backup process tree, start it from a systemd timer
  with `CPUQuota=`, `MemoryMax=` and `Nice=` in the service unit instead of cron.

## How it works

For each database:

1. `BACKUP DATABASE db TO File('<temporary directory>/<name>.tar') ... ASYNC` is started
   and polled through `system.backups`.
2. The size of `<name>.tar` is compared with the size reported by the server.
3. `<name>.tar` is compressed by `zstd` to `<name>.tar.zst` in the backup directory, then `<name>.tar` is removed.
   ClickHouse can write `.tar.zst` itself, but it does not allow to choose the compression level for tar archives.
4. The archive is flushed to the disk with `fsync` and dropped from the page cache.
5. The archive is verified: `zstd --test`, then `tar --list` must find the backup manifest `.backup` in it.
   Because of the previous step the archive is read from the disk, and not from the memory of the kernel.

The archives are written to the directory `<timestamp>-clickhouse-backup.partial`.
It is renamed to `<timestamp>-clickhouse-backup` only when all archives are created and verified,
so a directory without a suffix is always a complete backup.

### Durability

A file which was written but not flushed may stay only in the memory of the kernel for tens of seconds.
After a power failure such a file is truncated or empty. `zstd` does not flush its output, so clickhouse-backup
does it itself, in this order:

1. `fsync` of every archive, right after it is compressed;
2. `fsync` of the `CHECKSUM` file;
3. `fsync` of the backup directory: the names of the files in it become durable;
4. the directory is renamed from `.partial` to its final name;
5. `fsync` of `backup_root`: the new name of the directory becomes durable;
6. only then the success file is written (and flushed too), and old backups are removed.

So a directory with the name of a complete backup, a fresh success file and the removal of an old backup
always mean that the new backup is really on the disk.

Dropping an archive from the page cache before the verification is an advice to the kernel
(`posix_fadvise` with `POSIX_FADV_DONTNEED`). Local file systems such as XFS and ext4 follow it.
A file system with its own cache may ignore it, ZFS for example: the verification then reads the archive from
that cache, which is not an error. `fsync` of a directory is not supported by some network and FUSE file
systems: this is ignored, and the durability of the names then depends on that file system.

### Temporary files

Uncompressed tar archives are written to `<temporary_directory>/clickhouse-backup/<timestamp>-<random>/`,
`temporary_directory` is `/var/tmp` by default. Free space equal to the uncompressed size of the largest
database is required there.

#### Where to place the temporary directory

- **Keep it off the file system which holds the data of ClickHouse** (`/var/lib/clickhouse`), if you can.
  If the temporary directory shares a file system with the data and this file system becomes full
  during a backup, ClickHouse is unable to write: inserts and merges fail until the space is freed.
  On a separate file system only the backup fails. The same is true for `backup_root`.
- A separate device also makes the backup faster and lighter: `zstd` reads the tar archive from the temporary
  directory and writes the compressed archive to `backup_root`; on one device these reads and writes compete
  with each other and with the queries of the server.
- Do not use a `tmpfs` (`/tmp` is a `tmpfs` on many systems): the uncompressed archive would be kept in memory.
- `clickhouse-backup --check` shows whether the temporary directory and `backup_root` are on the same file system:
  the "free space" section then contains one line instead of two. Compare with `df /var/lib/clickhouse`.

#### Protection of the temporary files

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

Access through `/proc/<pid>/root` was tested with SELinux disabled only. With SELinux in the enforcing mode
it may be denied, depending on the domain in which clickhouse-backup is started: run `clickhouse-backup --check`
and one real backup after `PrivateTmp=yes` is enabled, and look for denials with `ausearch -m avc -ts recent`.

With `PrivateTmp=yes` systemd also mounts everything listed for `/tmp` and `/var/tmp` in `/etc/fstab` before
the service starts. Check where `/var/tmp` of the server really is, and how much space is there:

```
df -h /proc/$(systemctl show clickhouse-server -p MainPID --value)/root/var/tmp
```

### Free space

Before anything is started, clickhouse-backup compares the size of the databases with the free space
(the `free_space_check` key, `true` by default):

- the temporary directory must hold the uncompressed data of the largest database
  (of the `jobs` largest databases, if `jobs` is greater than 1);
- `backup_root` must hold the data of all databases, as if it were not compressible at all;
- 10% are added to both numbers;
- if the temporary directory and `backup_root` are on the same file system, the two numbers are added together
  and compared with the free space of this file system; if they are on different file systems,
  each number is compared with the free space of its own file system.

If there is not enough space, nothing is started, the server configuration is not touched, the exit code is `1`.
`clickhouse-backup --check` prints the required and the available space.

Limitations of this check:

- **The estimate for `backup_root` is intentionally too high.** The size of the compressed archives is not known
  in advance, so the worst case is assumed: no compression at all. Real archives are usually much smaller.
  On a disk which is almost full the check may therefore refuse a backup which would have fit.
  In this case free some space, reduce `backup_save_days`, or set `free_space_check = false`.
- **The check is made once, at the start.** It does not protect from a disk which is filled by something else
  while the backup is running, and it does not see the data inserted into the databases after the start.
  If the disk becomes full during the backup, the `BACKUP` statement or `zstd` fails, the backup of this
  database is counted as failed and its unfinished files are removed.
- With `free_space_check = false` nothing is checked: a backup which does not fit fails in the middle,
  after it has filled the disk. If this disk also holds the data of ClickHouse, inserts fail at that moment.

### Failures

- The backup of one database is cancelled and counted as failed if it takes more than `backup_timeout_hours`,
  `24` by default: the `BACKUP` query is killed on the server. Without this limit a hung backup
  would hold the lock forever.
- While the server executes a `BACKUP` statement, clickhouse-backup asks it for the status every few seconds.
  A busy server may refuse such a request (`Too many simultaneous queries`) or drop the connection.
  The backup itself is not affected, so the request is repeated. The backup is cancelled only if the status
  stays unknown for more than `backup_status_retry_seconds`, `300` by default, without a break.
- If the backup of at least one database fails, the directory is renamed to `<timestamp>-clickhouse-backup.failed`.
  It contains only the verified archives of other databases. The exit code is `1`.
- Complete backups older than `backup_save_days` are removed only after a successful run, so repeated
  failures never rotate the last good backups away.
- Only the newest `.failed` directory is kept, older ones are removed after any run: a failed backup is nearly
  as large as a complete one. The newest one is removed when it becomes older than `backup_save_days`.
- An old backup is renamed to `<timestamp>-clickhouse-backup.partial` before it is removed. If the removal
  is interrupted, the next run finishes it: a half-removed directory never has the name of a backup.
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

### Secrets in the backup

A backup directory contains secrets, protect it as carefully as the server itself:

- the `config-only` archive contains `/etc/clickhouse-server` with `users.xml` and `users.d`: passwords or
  password hashes of the users, and every other secret written to the configuration (credentials of
  dictionaries and of external storages, keys of encrypted disks, private keys of TLS certificates, if they
  are kept in this directory), and `/etc/clickhouse-client` with the passwords saved for the client;
- the `globals-only` archive contains the password hashes of the users created by SQL,
  and the named collections with their credentials;
- the archives of the databases contain the definitions of tables and dictionaries, which may include
  credentials of external sources.

`backup_root` and the directories of the backups have mode `0700` and belong to the `clickhouse` user,
the archives have mode `0640`. The archives are not encrypted. Every copy of this directory made by other tools
(file system snapshots, replication to another host, an external backup system) carries the same secrets:
check who is able to read them there.

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
   Or allow the directory by hand in a file with another name, for example
   `/etc/clickhouse-server/config.d/restore.xml`; the attribute keeps the paths of the main configuration allowed:
   ```
   <clickhouse>
       <backups>
           <allowed_path restore="1">/srv/clickhouse-backup/</allowed_path>
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

## Troubleshooting

All messages are printed to stderr and written to the journal: `journalctl -t clickhouse-backup --since today`.

Errors which clickhouse-backup foresees are reported as `BackupError` or `ConfigurationError` with a message which
says what is wrong. Any other error (`KeyError`, `JSONDecodeError`, `OSError` and so on) is unexpected: it is a bug
of clickhouse-backup or a change of the behaviour of ClickHouse. For such an error the journal also contains
the traceback, after the line `traceback of the unexpected error:`. It is not printed to stderr,
except in the `--check` mode.

| message or symptom | cause | what to do |
|--------------------|-------|------------|
| `another instance of clickhouse-backup is already running` | The previous run is still working or hangs. | `pgrep -af clickhouse-backup`, and look at the running backups: `SELECT name, status, start_time FROM system.backups WHERE status = 'CREATING_BACKUP'`. Send SIGTERM to a hung run: it kills its `BACKUP` query and cleans up. Never remove the lock file by hand. |
| `backup is not finished within backup_timeout_hours = N hours` | One database took longer than the limit, its `BACKUP` query was killed. | Raise `backup_timeout_hours`, or remove a too low `max_backup_bandwidth`. |
| `status of the backup is unknown for more than N seconds (backup_status_retry_seconds)` | The server did not answer the status requests for too long, the last error follows. The `BACKUP` query was killed. | Find out why the server refuses queries (overload, `max_concurrent_queries`, restart). Raise `backup_status_retry_seconds` if such periods are normal for this server. |
| `not enough free space in DIR: X required, Y available` | The estimate of the required space is higher than the free space. | Free some space or reduce `backup_save_days`. The estimate for `backup_root` assumes no compression: if you know that the backup fits, set `free_space_check = false`. See [Free space](#free-space). |
| `DIR must be a directory owned by the user clickhouse with mode 0700, but something else was found there` | A file, a symbolic link or a directory of another user occupies the name of the temporary directory. | Look at it (`ls -ld DIR`), find out who has created it, remove it. |
| `server configuration file FILE was changed, but the server has not applied it within N seconds` | The server has not reloaded its configuration. | `clickhouse-client --query "SYSTEM RELOAD CONFIG"`, then run the backup again. Look for errors of the configuration in `/var/log/clickhouse-server/clickhouse-server.err.log`. |
| `server configuration file FILE exists, but it was not written by clickhouse-backup` | Somebody has created a file with the name which clickhouse-backup uses. | Rename the file, or set another name in the `server_config_file` key. |
| `Authentication failed` or `AUTHENTICATION_FAILED` in the message | `clickhouse-client` has found no valid user name and password. | Check `clickhouse-client --query "SELECT 1"` as root. Set `clickhouse_client_config` to a file with the credentials, see [Connection to the server](#connection-to-the-server). |
| `Connection refused` in the message | The server is not running or does not listen on the address used by `clickhouse-client`. | `systemctl status clickhouse-server`. |
| `backup of NAME failed: ...` and a directory with the `.failed` suffix | The backup of one database has failed, the reason is in the message. The other archives in this directory are complete and verified. | Fix the reason and run the backup again. The archives of a `.failed` directory can be used for a restore. Complete backups are not rotated away while backups fail. |
| `backup status is BACKUP_FAILED: ...` | The `BACKUP` statement has failed inside the server, its error follows. | Read the error; the full text is in `SELECT error FROM system.backups ORDER BY start_time DESC LIMIT 1`. |
| `server reported a successful backup, but FILE does not exist` | The server and clickhouse-backup see different file systems, and the server was not detected correctly. | Check that the server runs on this host and that `<data path>/status` contains its PID. |
| `table 'DB'.'TABLE': data of the ENGINE engine is not saved to the backup` | A warning, not an error: `BACKUP` does not save data of this engine. | Nothing, if the data is not needed. Add the table to `exclude_tables` to silence the warning. See [What is saved, and what is not](#what-is-saved-and-what-is-not). |
| `unexpected path PATH` | A warning: `backup_root` contains something which is not a backup directory. | Remove or move it. |
| `removing directory ... left by an interrupted run`, `killing unfinished backup ...` | A warning: the previous run was killed or the host was rebooted during a backup. This run has cleaned up after it. | Nothing. |
| `backup of NAME creation time HH:MM:SS exceeds warning_threshold_hours limit` | A warning: the backup of this database is slower than expected. | Raise `warning_threshold_hours` for this database, or find out why it has become slower. |
| `TOML parser not found` | Python is older than 3.11 and the `tomli` module is not installed. | `python3 -m pip install tomli` |
| The file `last-successful-backup` is old, and there are no messages at all | The backup is not started: cron or the timer does not work. | `systemctl list-timers clickhouse-backup.timer` or `systemctl status crond`, then start the backup by hand. |

## License

GNU General Public License v3.0 or later, see the `LICENSE` file.

The list of changes of every version is at the beginning of the `clickhouse-backup` script.
