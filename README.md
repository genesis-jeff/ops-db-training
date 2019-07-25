# Mongodb Operation

---

## Requirements
* https://portainer.readthedocs.io/en/latest/deployment.html
* docker-compose version 1.23.2, build 1110ad01
* Docker version 18.09.2, build 6247962

---

## Standalone

- How to authenticate with mongodb (SCRAM), --authenticationDatabase 

- Initial admin via Local Exception
https://docs.mongodb.com/v3.4/core/security-users/#localhost-exception

```
db.createUser({ user: "admin", pwd: "adminpass", roles: [{ role: "root", db: "admin" }] })
```

---

- Mongodb URI format
https://docs.mongodb.com/manual/reference/connection-string/

```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/[database][?options]]
mongo "mongodb://admin:adminpass@localhost:27018/test?authSource=admin"
```

****
/database if specified: --authenticationDatabase/authSource=database,db=database
/database if not specified: --authenticationDatabase/authSource=admin,db=test
****

---

- View current operation
https://staroad.atlassian.net/wiki/spaces/ITOPS/pages/461373742/Locate+and+kill+long+running+queries+in+mongodb

```
db.currentOp()

```	
---

## Replica Set
https://docs.mongodb.com/v3.4/core/replica-set-architecture-three-members/#three-member-replica-sets


- Initiate a replicaset
https://docs.mongodb.com/manual/administration/replica-set-deployment/

```
rs.initiate(
  {
    _id: "myrs",
	members: [
	  { _id : 0, host : "rs0:27017" },
	  { _id : 1, host : "rs1:27017" },
	  { _id : 2, host : "rs2:27017" }
	]
  }
)

```

---

- Check status

```
rs.status()
rs.isMaster()
```
- Shutdown process

https://docs.mongodb.com/v3.4/tutorial/manage-mongodb-processes/

```
use admin
db.shutdownServer()

```

---

### Rolling Restart

- Ensure node is not primary 
```
rs.stepDown()
```
- Verify 
```
rs.isMaster()
```
- Shutdown node
```
db.shutdownServer()
```

- Start mongod process
```
mongod --port --shardsvr ...
```

---

- Force a member to be primary

https://docs.mongodb.com/v3.4/tutorial/force-member-to-be-primary/#force-a-member-to-be-primary-using-database-commands
```
rs.freeze()
```

---

- Handling stale secondary 

https://docs.mongodb.com/v3.4/tutorial/resync-replica-set-member/index.html#resync-a-member-of-a-replica-set

> [rsBackgroundSync] too stale to catch up -- entering maintenance mode


---

 * Shutdown stale node

```
db.shutdownServer()
```

 * Delete data folder.

```
rm -rf /data/db/*
```

 * Start mongod process

```		
mongod --port --shardsvr
```		
 
 * View oldest oplog entry

```
rs.printReplicationInfo()
```

 * Generate data

```
docker exec -it java bash
java -jar /java/POCDriver.jar -k 20 -i 10 -u 10 -b 20 -c "mongodb://172.23.0.4,172.23.0.5,172.23.0.6:27017/?replicaSet=myrs"
```

---

### What if OPLOG is not enough 

- Resize Oplog? 

- Volume snapshot.

---

### WriteConcern and Journal (v3.4.10)

- Defaults
  - w:1 (primary only)
  - j: unspecified (j:false)

- COLO: journal is enabled
  
---

## Sharded cluster
 https://docs.mongodb.com/v3.4/tutorial/deploy-shard-cluster/index.html#deploy-a-sharded-cluster

- Summary: 

 Initiate config replica set (--configsvr)
 
 Initiate shard(s) replica set (--shardsvr)
 
 Add shards thru mongos
 
```
docker-compose -p shard -f mongodb-shard-cluster.yml up
```

---

- Deploy config replica set

```
docker exec -it conf0 bash
mongo localhost:27019
rs.initiate(
  { 
    _id: "cnf0",
    configsvr: true,
    members: [ 
      { _id : 0, host : "conf0:27019" },
      { _id : 1, host : "conf1:27019" },
      { _id : 2, host : "conf2:27019" } 
    ] 
  } 
)
```

---

- Deploy shards

```
docker exec -it rs0 bash

mongo localhost:27018
rs.initiate(
  { 
    _id: "shard0",
    members: [ 
      { _id : 0, host : "rs0:27018" },
      { _id : 1, host : "rs1:27018" },
      { _id : 2, host : "rs2:27018" } 
    ] 
  } 
)
```

---

- Add shards

```
docker exec -it mongos0 bash
mongo localhost:27017
 
sh.addShard("shard0/rs0:27018")
```

- Verify

```
sh.status()
```

- Generate Data

```
docker exec -it java bash
java -jar /java/POCDriver.jar -k 20 -i 10 -u 10 -b 20 -w -c "mongodb://mongos0"
```

---

- Shard Distribution

```
use POCDB

mongos> db.POCCOLL.getShardDistribution()

Shard shard0 at shard0/rs0:27018,rs1:27018,rs2:27018
 data : 42.73MiB docs : 151717 chunks : 5
 estimated data per chunk : 8.54MiB
 estimated docs per chunk : 30343
```

---

## Shard a collection
https://docs.mongodb.com/v3.4/tutorial/deploy-sharded-cluster-hashed-sharding/

- Create hashed index

```
db.POCCOLL.createIndex({_id: 'hashed'})
```

- Enable sharding on database

```
sh.enableSharding("POCDB")

```

- Shard collection using hashed key

```
sh.shardCollection("POCDB.POCCOLL", { _id : "hashed" } )
```

---

### Balancer Management
```
sh.getBalancerState()
sh.setBalancerState({true/false})
```

---

- Create another shard (shard1)

```
docker-compose -p shard -f mongodb-shard-cluster.yml up rs3 rs4 rs5
```


```
docker exec -it rs3 bash

mongo localhost:27018
rs.initiate(
  { 
    _id: "shard1",
    members: [ 
      { _id : 0, host : "rs3:27018" },
      { _id : 1, host : "rs4:27018" },
      { _id : 2, host : "rs5:27018" } 
    ] 
  } 
)
```

- Add shard1 to mongos

```
docker exec -it mongos0 bash
mongo localhost:27017

sh.addShard("shard1/rs3:27018")
```


