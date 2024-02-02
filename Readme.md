
# Ansible role install and configure Restic backup

Description

-----------
Installs restic on client without running restic as root rather it creates a user that will run restic backups.
This role allows scheduled backups via systemd.
The Restic executable will have a capability to read any file on system regardless of its permissions.

> NOTE: this role requires systems with systemd

verifyed supported platforms:

- Ubuntu: 20.04, 21.04, 22.04, 20.10, 21.10, 22.04.
- Debian: all

## Available variables

### ***Basic installation***

| Name               | Description  |
| ------------------ | ------------ |
| **paths** (list) | list of paths to backup
| **password** (string) | password for accessing repository (required by restic)
| **storage** (dict) | dict which must contain `repository` key with URL to repository (see `RESTIC_REPOSITORY` in restic docs). All other keys will be converted to upper case and passed as env variables
|restic_version: | release version which must be installed
|restic_install_dir:| path to where restic binary is installed
|restic_conf_dir:| directory to store configuration profiles for backups
|restic_user: | name of user to run restic and own all configuration
|restic_home: | home dir for `restic_user`
|restic_exports: | includes all env variables to connect to s3 storage and init repository

See [defaults/main.yml](./defaults/main.yml) for more details

### ***Advanced usage with scheduler***

| Parameter | Description |
| --------- | ----------- |
| **name** (string) | name for the profile which will be used in service/timer names: e.g. `restic@<name>.service` and `restic@<name>.timer`
| storage.repository2 (string) | secondary repository for snapshot replication
| stdin (list of objects) | {name: "filename.out", command: "echo 'Test data'"}
| stdin.name (string) | file name for command the output
| stdin.command (string) | command to run and get the output from
| stdin.tags (list) | extra tags for this backup command
| schedule (string or dict) | specify custom schedule string which is passed to [systemd.timer](https://www.man7.org/linux/man-pages/man7/systemd.time.7.html#CALENDAR_EVENTS) `OnCalendar` parameter. By default `restic_systemd_timer_default_oncalendar` is used
| schedule.check (string) | specify different schedule for repository consistency check
| schedule.prune (string) | specify different schedule for deleting osolete snapshots from repository
| schedule.backup (string) | specify backup schedule
| keep (dict) | dict with keep policy for `restic forget` command which is executed after backup. Can have following keys: `last`, `daily`, `weekly`, `monthly`, `yearly` (see [docs](https://restic.readthedocs.io/en/stable/060_forget.html#removing-snapshots-according-to-a-policy))
| exclude (list) | list of exclude patterns for `--exclude` flags (see [docs](https://restic.readthedocs.io/en/stable/040_backup.html#excluding-files))
| tags (list) | list of tags for backup used for both `restic backup` and `restic forget` commands. By default tags from `restic_default_tags` variable are applied
| options (dict) | dict with extra options passed to backup command with `-o` flag
| extra_flags (list) | list of any extra flags to pass to `restic backup` command
| check_flags (list) | list of any extra flags to pass to `restic check` command
| prune_flags (list) | list of any extra flags to pass to `restic prune` command
|restic_timer_randomized_delay| randomized delay sec for systemd timer
|restic_backup_schedule| value for OnCalendar option in systemd timer used for backup tasks by default
|restic_prune_schedule| value for OnCalendar option in systemd timer used by restic-prune tasks by default
|restic_check_schedule| value for OnCalendar option in systemd timer used by restic-check tasks by default
|restic_copy_schedule| value for OnCalendar option in systemd timer used by restic-copy tasks by default
|restic_create_schedule| whether to add or delete schedules for backup
|backup_dir:| which directory to backup |
> Parameters in **bold** are required, others are optional.

## Example playbook

```yaml
restic_user: backup
restic_profiles:
- name: main
  paths: 
    - /etc
    - /home
    - /var/lib
  exclude:
    - "/home/*/.local"
    - "/home/*/.cargo*"
    - "/home/*/.gem*"
    - "/home/*/.npm*"
    - "/home/*/.ccache*"
    - "*/.cache*"
  extra_flags:
    - "--one-file-system"
  stdin:
    #example generic stdout
    - name: package_list.txt
      tags:
      - apt
      command: apt list --installed
    # mysqldum from docker container
    - name: kimai.sql
      tags: 
      - docker
      - sql
      command: docker compose -f /srv/kimai/docker-compose.yml exec -T sqldb sh -c 'mysqldump -p$MYSQL_ROOT_PASSWORD --single-transaction --default-character-set=utf8mb4 --all-databases'
  password: pwd!
  keep:
    last: 5
    daily: 7
    weekly: 9
    monthly: 12
    yearly: 3
  storage:
    repository: /backup/restic_main
  schedule:
    backup: '*-*-* 02:00:00'
    prune:  'Sat *-*-* 04:00:00'
    check:  'Sun *-*-01..07 02:00:00'
  check: all
restic_systemd_timer_randomizeddelaysec: '4h'
restic_systemd_timer_default_oncalendar: '*-*-* 02:00:00'
restic_create_schedule: true
```