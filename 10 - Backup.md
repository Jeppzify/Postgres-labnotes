# Continuous backup

## Method 1 - pg_dumpall

### Running a script with cron on backup-server.
Create a pgpass-file in home-dir containing the connection-string.
```bash
vim ~/.pgpass
"server1:5432:*:postgres:{password}"
```

Create a bash-script to connect to server1 and dump all DB's.
Compress with gzip directly from stdout without creating intermediary files to save space on disk.
```bash
vim backup.sh
#!/bin/bash
fileName="backup_$(date +%F_%H:%M).sql.gz";
pg_dumpall -h server1 -U postgres | gzip > /postgres/backups/$fileName
```

Create a file in cron.d to run the backup every 15 mins and then remove dumps older than one hour.
```bash
/etc/cron.d# cat sql-backup 
*/15 * * * * postgres /var/lib/postgresql/backup.sh; find /var/lib/postgresql/backups/ -mmin +59 -type f -delete

~/backups$ ls
backup_2021-02-17_21:10.sql.gz
```


## Method 2 - Log-shipping (wal-shipping) 
### From server1 to backup-server with rsync

Set up ssh with keys.
On backup-server:
```bash
ssh-keygen -f backup.key
mv backup.key.pub > authorized_keys
chmod 600 authorized_keys 
```

Transfer the private key to server1 with any file-transfer method

On server1:
```bash
chmod 600 ~/.ssh/backup.key
vim ~/.ssh/config
Host backup
     HostName backup-server
     User postgres
     IdentityFile ~/.ssh/backup.key

ssh backup
```
Type 'yes' first time at prompt to add the remote host to 'known_hosts'.
The next time it will connect without any prompt.
```bash
"Welcome to Ubuntu 20.04....."
postgres@backup-server:~$
```

Now server1 can automatically connect with rsync to the backup-server.

### Enable wal-archiving
```bash
vim /etc/postgresql/9.3/main/postgresql.conf
wal-level = archive
archive-mode = on
archive-command = rsync -a %p backup-server:~/12/archive/%f
pg_ctl reload -D /var/lib/postgresql/9.3/main
```
Do a manual base-backup
```bash
pg_basebackup > base-backup
rsync -a base-backup backup-server:~/12/archive/base
```

Now the WAL-files are written and copied to the backup server and can be used to restore 

## Method 3 - Streaming Replication

### Streaming to a clone of server1 (referred to as 'server2') which acts as a warm standby server

Setup config on server1:
```bash
vim /etc/postgresql/9.3/main/postgresql.conf
 - wal_level = hot_standby
 - max_wal_senders = 10
 - wal_keep_segments = 50
 - hot_standby = on
```
Create a new user and database for replication inside the main cluster
```bash
vim /etc/postgresql/9.3/main/pg_hba.conf
 - host repuser     repuser 192.168.1.142 md5
 - host replication repuser 192.168.1.142 md5
```
Do a basebackup on server2 and create 'recovery.conf' file before starting postgres
```bash
cd ~/9.3/main
rm *
pg_basebackup -h server1 -p 5432 -U repuser -w -R -D .
```
The '-R' flag generates a recovery-file but it may need a few modifications.
In my case I had to add 'password=[password]' to primary_conninfo in the file to make the connection work.
```bash
cat recovery.conf
 - standby_mode = 'on'
 - primary_conninfo = 'user=repuser password=[password] host=server1 port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres'
pg_ctl start -D .
```

Checking processes with 'ps -ef'
```bash
/usr/lib/postgresql/9.3/bin/postgres -D /var/lib/postgresql/9.3/main -c config_file=/etc/postgresql/9.3/main/postgresql.conf
startup process   recovering 000000020000000000000019
checkpointer process
writer process
wal receiver process   streaming 0/19001850
```
Checking replication stat on server1
```sql
psql
select * from pg_stat_replication;
pid  | usesysid | usename | application_name |  client_addr  | client_hostname | 
5503 |    18898 | repmgr  | walreceiver      | 192.168.1.142 |                 | 
------+----------+---------+------------------+---------------+-----------------+---------
client_port | backend_start                 | state     | sent_location | write_location | 
36236       | 2021-03-01 22:08:01.154524+01 | streaming | 0/19001990    | 0/19001990     | 
------+----------+---------+------------------+---------------+-----------------+---------
flush_location | replay_location | sync_priority | sync_state 
0/19001990     | 0/19001990      | 0 | async
------+----------+---------+------------------+---------------+-----------------+---------
```

Should the main server crash it is now possible to switch over to the clone instead. Failover can, and should, be automated to minimize downtime.
I will explore this later.

# Backup and recovery with barman 

Barman is a tool that works just like when we set up log-shipping before but barman has some nice additional features like automatic recovery script creation and better backup management.

Install barman on the backup-server
```bash
sudo apt-get install barman barman-cli
passwd barman
```
Set up trusted ssh connection with keys between the servers like we did before

Configuring barman
There are a lot of options, I changed the following:
```
vim /etc/barman.conf
compression = gzip
reuse_backup = link
backup_method = rsync
archiver = on
```

Create a configuration file for the postgres server
```bash
vim /etc/barman.d/server1
[server1]
ssh_command = ssh server1
conninfo = host=192.168.1.162 port=5432 user=postgres password=[password]
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
```

Use the command "barman show-server server1" to see all config parameters
Change the archive command in postgresql.conf on server1 and point it to the full path of barmans 'incoming_wals_directory' which is displayed in the show-server command output
```bash
archive_command = ‘rsync -a %p barman:/var/lib/barman/server1/incoming/%f'
```

Restart postgres

Check if everything is set up correct and start backup
```bash
barman check server1
Server server1:
    PostgreSQL: OK
    is_superuser: OK
    wal_level: OK
    directories: OK
    retention policy settings: OK
    backup maximum age: FAILED (interval provided: 1 day, latest backup age: No available backups)
    compression settings: OK
    failed backups: OK (there are 0 failed backups)
    minimum redundancy requirements: OK (have 0 backups, expected at least 0)
    ssh: OK (PostgreSQL server)
    not in recovery: OK
    archive_mode: OK
    archive_command: OK
    archiver errors: OK

barman backup server1
barman list-backup server1
    server1 20210223T010520 - Tue Feb 23 01:05:33 2021 - Size: 96.2 MiB - WAL Size: 5.0 MiB
```
Add a job to run backup every 4 hours with barman cron by editing crontab
```bash
* * * * * /usr/bin/barman cron
0 */4 * * * /usr/bin/barman backup server1
```
### Test recovery
Stop postgres on server1.
Show backup details and start the recovery.
```bash
barman show-backup server1 20210223T010520
barman recover --remote-ssh-command "ssh server1" --target-time="2021-02-23 01:05:33.606425+01:00" server1 20210223T010520 /var/lib/postgresql/9.3/main
```

The file 'recovery.conf' is automatically created by barman in the data-dir
Log in to server1 and run:
```bash
pg_recover recovery.conf
```

Check all configs and data to make sure everything got restored!
