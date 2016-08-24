#### mnmongoinact2ed
#####Chapter 10. WiredTiger and pluggable storage
wiredTiger config
```
storage:
  dbPath: "/data/ke"
  journal:
    enabled: true
  engine: "wiredTiger"
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
      journalCompressor: none
    collectionConfig:
      blockCompressor: none
    indexConfig:
      prefixCompression: false
```
######10.2.2. Migrating your database to WiredTiger
create a mongodump
```
mkdir ~/dump
cd ~/dump
mongodump
```
then, move the mmapv1-based data away
```
mv /data/ke /data/mmapv1
```

create a fresh dict for new restore.
```
mkdir /data/ke && chown mongodb:mongodb /data/ke && chmod 755 /data/ke
cd ~/dump
mongorestore dump
```
######10.3.1. Configuration files
mmapv1
```
storage:
  dbPath:"./data/mmapv1"
  directoryPerDB: true
  journal:
    enabled: true
  systemLog:
    destination: file
    path: "./log.log"
    logAppend: " true"
    timeStampFormat: iso8601-utc
  net:
    bind: 127.0.0.1
    port: 27017
    unixDomainSocket:
      enabled: true
```
