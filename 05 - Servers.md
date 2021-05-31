# Master server - Installation and setup - Postgresql 9.3

## Ubuntu 14.04, hostname 'server1'

```bash
apt-get update
apt-get install postgresql (optional: pgadmin3 postgresql-contrib) 
```

Postgresql 9.3 is auto-configured and started locally upon installation with user and db 'postgres' in cluster 'main'.

Lets set password for user and add bin-folder to path system-wide
```bash
sudo passwd postgres
sudo vi /etc/environment
PATH=...":/usr/lib/postgresql/9.3/bin"
su postgres
psql postgres
-- \password postgres
```


### Open up DB for connections on LAN
```bash
vim /etc/postgresql/9.3/main/pg_hba.conf
TYPE  DB     USER      ADDRESS         METHOD
host  all    postgres  192.168.1.1/24  md5

vim /etc/postgresql/9.3/main/postgresql.conf
listen_addresses = '*'
```

Restart the cluster and make sure postgres is listening.
```bash
pg_ctl reload -D /var/lib/postgresql/9.3/main

netstat -alnp | grep 5432
tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN      5885/postgres 
tcp6       0      0 :::5432                 :::*                    LISTEN      5885/postgres 
```

Add a few sample DB's found on github for substance.
https://github.com/morenoh149/postgresDBSamples.git

createdb [DBNAME]
psql -f [FILENAME]


# Backup server - Postgresql 12.5

## Ubuntu 20.04, hostname 'backup-server'
Mostly the same installation-process as for 9.3 but with local access only and no new DB's.

### Test connection to server1
```bash
psql -h server1 -U postgres 
Password for user postgres: 
psql (12.5 (Ubuntu 12.5-0ubuntu0.20.04.1), server 9.3.24)
Type "help" for help.

postgres=# 
```

Success!

