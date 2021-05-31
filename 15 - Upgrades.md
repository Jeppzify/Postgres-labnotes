# Upgrading PostgreSQL (work in progress)


## Method 1 - incremental with do-release-upgrade and pg_upgradecluster 

### Pros: Safe, single server, no need for additional configuration
### Cons: Time-consuming and requires several reboots/downtime

```bash
sudo apt update (&& sudo apt upgrade/dist-upgrade)
sudo do-release-upgrade
pg_dropcluster 9.5 main --stop
pg_ctlcluster stop 9.3 main
pg_upgradecluster 9.3 main 
```
Use pg_upgradecluster with -k to link the clusterfiles to the new version instead of making copys if storage is limited.

Make sure 9.5 is running, check connection and data. If everything is fine we can remove the previous version
```bash
pg_dropcluster 9.3 main
sudo apt purge postgresql-9.3
```

Repeat the same steps for versions 10 > 11 > 12 (>13)

## Method 2 - From 9.3 to 12.5 using sql-dump

### Pros: Easy, single period of downtime
### Cons: Requires a second server, total downtime might be longer

Restrict access to master to prevent apps from making changes
Use pg_dumpall 
On postgres 12, psql -f dumpfile
Check data, shut down old server
Change IP, hostname etc. on the new server
Restore access 
Restart postgres