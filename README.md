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

## Config File First!

No matter how you use it, you need to setup the configuration file, conveniently provided here in the repo for you, with examples!

The script will look in three places for it:

```bash
/etc/run-mariabackup.conf
/etc/run-mariabackup/run-mariabackup.conf
~/.config/run-mariabackup.conf
```

the last option being the one I personally prefer using, especially if it contains passwords, with proper flags set on the file.

Also, the actual `mariabackup` commands in the script are run using `sudo`, so it can not only traverse to your home directory, but it'll bypass any other weirdness that can crop up. So, you're welcome to `chmod 600` your config file in your home directory.

## Command Line

Your first time running this? From within the git repo, run:

```bash
./run-mariabackup
```

And it'll auto-link/auto-install into `/usr/local/bin` like most of my bash scripts do. This way, the updating process is running `git pull` from the repo.

## Crontab

```bash
cat << EOF | sudo tee /etc/cron.d/run-mariabackup
#MariaBackup

30 2 * * * /usr/local/bin/run-mariabackup > /var/log/run-mariabackup.log 2>&1
EOF
```

Thats a generic crontab that runs everyday at 2:30 am

* * *

## Restore Example

This is essentially what your backup directory will look like:

```bash
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
```

todo: I will be making a restore part of the script in the future.... just fyi

To restore your backups:

```bash
# decompress
cd /your/backup/directory

# Do a manual search for yourself first, to find the correct timestampted
# directory to use here. Then, put it down as the DATE variable.
DATE="2018-10-23_10-07-31"

# The DIR variable becomes the --target-dir below
while IFS= read -r line; do
    DIR="${line}/backup"
    mkdir -p "${line}/backup"
    zcat "${line}/backup.stream.gz" | mbstream -x -C "${line}/backup/"
done < <(find . -iname backup.stream.gz | grep "${DATE}" | xargs dirname)

# Prepare the files

mariabackup --prepare --target-dir "${DIR}" --user backup --password "YourPassword" --apply-log-only

mariabackup --prepare --target-dir "${DIR}" --user backup --password "YourPassword" --apply-log-only --incremental-dir incr/2018-10-23_10-07-31/2018-10-23_10-08-49/backup/

mariabackup --prepare --target-dir "${DIR}" --user backup --password "YourPassword" --apply-log-only --incremental-dir incr/2018-10-23_10-07-31/2018-10-23_10-13-58/backup/

# stop mariadb

sudo systemctl stop mariadb

# empty datadir

mv /data/mysql/ /data/mysql_bak/

# copy-back

mariabackup --copy-back --target-dir "${DIR}" --user backup --password "YourPassword" --datadir /data/mysql/

# fix privileges

chown -R mysql:mysql /data/mysql/

# start mariadb

service mariadb start

# done!

```
