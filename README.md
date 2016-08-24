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
wiredTiger - uncompressed - same as previous demo
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
snappy - change blockCompressor and prefixCompression
```
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```
zlib: same
```
    collectionConfig:
      blockCompressor: zlib
    indexConfig:
      prefixCompression: true
```
######10.3.2. Insertion script and benchmark script
review: array and seq and echo -ne
```
#!/bin/bash

export MONGO_DIR=/storage/mongodb
export NUM_LOOPS=16

configs=(
    mmapv1.conf
    ke1.conf
    ke2.conf
    ke3.conf
)
cd $MONGO_DIR
for config in "${configs[@]}"; do
  DATA_DIR=$(grep dbPath configs/$config | awk -F\" '{ print $2 }')
  echo -ne "Starting up mongod... "
  T="$(date +%s)"
  ./bin/mongo admin --quiet --eval "db.shutdownServer({force: true})" >/dev/null
  SIZE=$(du -s --block-size=1 $MONGO_DIR/$DATA_DIR | cut -f1)
  SIZE_MB=$(echo "scale=2; $SIZE/(1024*1024)" | bc)
  echo "Disk usage for $config: ${SIZE_MB}MB"2>&1
done
```

######Chapter 11. Replication
create two nodes and 1 arbiter
```
mkdir ~/node1
mkdir ~/node2
mkdir ~/arbiter
mongod --replSet myapp --dbpath ~/node1 --port 40000
mongod --replSet myapp --dbpath ~/node2 --port 40001
mongod --replSet myapp --dbpath ~/arbiter --port 40002
mongo --port 40000
rs.initiate()
rs.add("localhost:40001")
rs.addArb("localhost:40002") //3.0+
db.isMaster() //get summary
rs.status() //more detailed
```
if querying in secondary, must run slaveOk() first
```
rs.slaveOk()
```
one time slaveOk execution  
[see here](http://stackoverflow.com/questions/33366182/how-to-set-rs-slaveok-in-secondary-mongodb-servers-in-replicaset-via-commandli)
```
mongo.exe --port 40000 --eval "rs.slaveOk()" --shell
```

leave current node from replSet
```
mongo --port 40000
use admin
db.shutdownServer()
```
alternative method: 1: ctrl+C 2: If run in bkg,
```
ps aux | grep mongod.lock
kill -3
```
