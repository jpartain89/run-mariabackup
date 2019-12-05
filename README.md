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

For the `${DATE}` info that we are going to be using as the base of our directory listings, that would be the date info for the original, full backup you performed that is inside the `/base/` directory.

```bash
# decompress
cd /your/backup/directory

# For the DATE info that we are going to be using as the base of our
# directory listings, that would be the date info for the original, full
# backup you performed that is inside the `/base/` directory.
DATE="2018-10-23_10-07-31"

# The DIR variable becomes the --target-dir below
while IFS= read -r line; do
    mkdir -p "${line}/backup"
    zcat "${line}/backup.stream.gz" | mbstream -x -C "${line}/backup/"
done < <(find . -iname backup.stream.gz | grep "${DATE}" | xargs dirname)

# Prepare the files
# The --target-dir is the original, base directory.
mariabackup --prepare --target-dir ./base/${DATE}/backup/ --user mariabackup --password "YourPassword" --apply-log-only

# Now, we add on the incremental backup directories, running this command
# however many times you have incremental backups saved.
mariabackup --prepare --target-dir ./base/${DATE}/backup/ --user mariabackup --password "YourPassword" --apply-log-only --incremental-dir incr/${DATE}/2018-10-23_10-08-49/backup/

mariabackup --prepare --target-dir ./base/${DATE}/backup/ --user mariabackup --password "YourPassword" --apply-log-only --incremental-dir incr/${DATE}/2018-10-23_10-13-58/backup/

# Next, we stop mariadb before we do the restoration
sudo systemctl stop mariadb

# Find where your datadir is saved at:
# Debian default is /var/lib/mysql
# You can look in your config files in /etc/mysql to locate this dir.
mv /data/mysql/ /data/mysql_bak/

# Now, we will copy the restored files to their final working spot
# MariaBackup lets you either copy the files or move the files
mariabackup --copy-back --target-dir ./base/${DATE}/backup/ --user mariabackup --password "YourPassword" --datadir /data/mysql/

# Lastly, once everything is in place, we want to make sure the
# permissions are set correctly.
chown -R mysql:mysql /data/mysql/

# Lastly, start it back up again!
sudo systemctl start mariadb.service

# done!
```
