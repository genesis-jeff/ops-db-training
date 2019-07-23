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

- Check status
```
rs.status()
```

- Shutdown process
https://docs.mongodb.com/v3.4/tutorial/manage-mongodb-processes/
```
use admin; db.shutdown()
```

- Rolling Restart
  - Ensure node is not primary 
```
rs.stepDown()
```
  - Verify 
```
rs.isMaster()
```
  - sdfsd
---
- Rolling Restart
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

- Handling stale secondary
https://docs.mongodb.com/v3.4/tutorial/resync-replica-set-member/index.html#resync-a-member-of-a-replica-set

> [rsBackgroundSync] too stale to catch up -- entering maintenance mode

---

  - Shutdown stale node
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
java -jar POCDriver.jar -k 20 -i 10 -u 10 -b 20 -c "mongodb://172.23.0.2,172.23.0.3,172.23.0.4:27018/?replicaSet=shard0"
```

---

## Sharded cluster
https://docs.mongodb.com/v3.4/tutorial/deploy-shard-cluster/index.html#deploy-a-sharded-cluster

Summary: 

 - Deploy config replica set
 - Deploy shard(s)
 - Add shards thru mongos


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
