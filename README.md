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
######11.2.2. How replication works
rs.oplog
```
use local
show collections
db.oplog.rs.findOne({op: "i"})
```
changed in 2,6, [update api](https://docs.mongodb.com/manual/reference/method/db.collection.update/)
```
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)
```
check u(update) type of oplog
```
db.oplog.rs.find({op: "u"})
```

get latest entry
```
db.oplog.rs.find().sort({$natural: -1}) .limit(1)
```
some basic info about current status oplog
```
db.oplog.rs.find().sort({$natural: -1}) .limit(1)
```
assign oplogSize (1024MB)
```
mongod --replSet myapp --oplogSize 1024
```
######Heartbeat and failover
rs.status() ->1 healthy,0 unresponsive  
When the primary can’t see a majority, it must step down.(become secondary)  

######Commit and rollback
In a rollback, all writes that were never replicated to a majority are undone.  
They’re removed from both secondary’s oplog and the collection where reside.  
The reverted writes are stored in the rollback subdirectory.Check using
```
bsondump
```
######11.2.3. Administration
config details.
```
settings.getLastErrorModes 
settings.getLastErrorDefaults
```

lets start.
```
config = {_id: "ke", members: []}
config.members.push({_id: 0, host: 'localhost:40000'})
config.members.push({_id: 1, host: 'localhost:40001'})
config.members.push({_id: 2, host: 'localhost:40002', arbiterOnly: true})
db.runCommand({replSetInitiate: config});
```
Replica set states  
GOOD state= 1,2,7  , or check with
```
rs.status()
replSetGetStatus //js
```
######Failure modes and recovery
