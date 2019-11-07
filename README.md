# Run-MariaBackup

_The original is forked from [jmfederico/run-xtrabackup.sh](https://gist.github.com/jmfederico/1495347)_

_Mine is forked from [omegazeng/run-mariabackup](https://github.com/omegazeng/run-mariabackup)_

## TL;DR
### Links:

[Full Backup and Restore with Mariabackup](https://mariadb.com/kb/en/library/full-backup-and-restore-with-mariabackup/)

[Incremental Backup and Restore with Mariabackup](https://mariadb.com/kb/en/library/incremental-backup-and-restore-with-mariabackup/)

[Installing MariaBackup](https://mariadb.com/kb/en/library/mariabackup-overview/#installing-mariabackup)

* * *

## Installing Mariabackup

### Debian-based OS:

```bash
sudo apt install mariadb-backup
```

### Or,

follow along with [MariaDB's instructions...](https://mariadb.com/kb/en/library/mariabackup-overview/#installing-mariabackup)

## Create a backup user

I personally just use the backup user that mariaDB says to use in [their instructions:](https://mariadb.com/kb/en/library/mariabackup-overview/#authentication-and-privileges)

```sql
CREATE USER 'mariabackup'@'localhost' IDENTIFIED BY 'mypassword';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'mariabackup'@'localhost';
FLUSH PRIVILEGES;
```

# Usage

No matter how you use it, you need to setup the configuration file, conveniently provided here in the repo for you, with examples!

The script will look in three places for it:

```bash
/etc/run-mariabackup.conf
/etc/run-mariabackup/run-mariabackup.conf
~/.config/run-mariabackup.conf
```

the last option being the one I personally prefer using, especially if it contains passwords, with proper flags set on the file.

Also, the actual `mariabackup` commands in the script are run using `sudo`, so it can not only traverse to your home directory, but it'll bypass any other weirdness that can crop up.... 

## Crontab

    #MySQL Backup
    30 2 * * * MYSQL_PASSWORD=YourPassword bash /data/script/run-mariabackup.sh > /data/script/logs/run-mariabackup.sh.out 2>&1

* * *

## Restore Example

    tree /data/mysql_backup/
    /data/mysql_backup/
    ├── base
    │   └── 2018-10-23_10-07-31
    │       ├── backup.stream.gz
    │       └── xtrabackup_checkpoints
    └── incr
        └── 2018-10-23_10-07-31
            ├── 2018-10-23_10-08-49
            │   ├── backup.stream.gz
            │   └── xtrabackup_checkpoints
            └── 2018-10-23_10-13-58
                ├── backup.stream.gz
                └── xtrabackup_checkpoints

    ```bash

# decompress

  cd /data/mysql_backup/
  for i in $(find . -name backup.stream.gz | grep '2018-10-23_10-07-31' | xargs dirname); \
  do \
  mkdir -p $i/backup; \
  zcat $i/backup.stream.gz | mbstream -x -C $i/backup/; \
  done

# prepare

  mariabackup --prepare --target-dir base/2018-10-23_10-07-31/backup/ --user backup --password "YourPassword" --apply-log-only
  mariabackup --prepare --target-dir base/2018-10-23_10-07-31/backup/ --user backup --password "YourPassword" --apply-log-only --incremental-dir incr/2018-10-23_10-07-31/2018-10-23_10-08-49/backup/
  mariabackup --prepare --target-dir base/2018-10-23_10-07-31/backup/ --user backup --password "YourPassword" --apply-log-only --incremental-dir incr/2018-10-23_10-07-31/2018-10-23_10-13-58/backup/

# stop mairadb

  service mariadb stop

# empty datadir

  mv /data/mysql/ /data/mysql_bak/

# copy-back

  mariabackup --copy-back --target-dir base/2018-10-23_10-07-31/backup/ --user backup --password "YourPassword" --datadir /data/mysql/

# fix privileges

  chown -R mysql:mysql /data/mysql/

# start mariadb

  service mariadb start

# done!

```

```
