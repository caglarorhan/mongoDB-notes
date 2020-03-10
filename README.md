**MongoDB Notes**

> **IMPORTANT:** Notes are highly personal, irregular, unordered and may seems meaningless and confusing. 

- MongoDB has an _id property on every collection and document.
- This id created by mongoDB driver not mongoDB itself. This gets speed. 
- The id is made of 12 bytes 
  - 4 bytes : timestamp
  - 3 bytes : machine identifier
  - 2 bytes : process identifier
  - 3 bytes : counter
  
    ``1 Byte = 8 bit``
    
    ``8 bit stores 2^8 = 256 numbers ``
    
    ``2^8^3 = >16M numbers (almost unique)`` 

This means same second, same machine, same process identifier there may more than 16M numbers can be sit. Not 100% unique but almost. 

**RDBMS** systems has real unique because they increase id numbers by one on every new record. But this has some disadvantages.

MongoDB driver has speed and concurrency.

- You can ask for an id from mongoDB driver with ``mongoose``


    const mongoose = require('mongoose');
    const id = new mongoose.Types.ObjectId();
    console.log(id);
    //and we can extract time stamp from this id
    console.log(id.getTimestamp());
    // Also we can check if id is valid.
    const isValid = mongoose.Types.ObjectId.isValid('sampleid');
    console.log(isValid);


---
**Replication and Setting Up a Shardered Cluster**

- **Setting up a Replica Set**
 
    MongoDB stores data where you set it on the config file (for sample conf file look below)
    - Gain admin (root) privileges on target folders and subfolders 
        - Use `chown groupname:username /folder/path/`
    - Create folders where do you want to set your all database storages
        - Use `mkdir -p /folder/path/`
    - Create config files (.conf) 
        -Use `nano config-file.conf` (or `nano mongod-1.conf` etc.)
    - Create key file for secure connections between nodes 
        - Use `openssl rand -base64 741 > /folder/path/nameof-keyfile`
    - Start mongo daemon with `mongod -f mongod-1.conf`
    - Start/run any odd number of mongod nodes. Odd is required to election (voting) for Primer. (3, 5, 7, 9, etc)
    - Connect first node you create with `mongo --port <given port on conf file>`  
    - Initiating the replica set (replica sets name retrieved from conf files)
        - `rs.initiate()`  
    - Create user with first time local exception access (with chosen user name and password)
    
            use admin
            db.createUser({
              user: "m103-admin",
              pwd: "m103-pass",
              roles: [
                {role: "root", db: "admin"}
              ]
            })
    - Exit mongo shell with `exit` and connect to entire replica set (any node's ip with port number)
        - `mongo --host "m103/192.168.103.100:27001" `   
        - after login you can check replica set status with `rs.status()` or get the overview of replica set topology with `rs.isMaster()`
    - To add other nodes (members) to this replica set continue with:
        - `rs.add("m103:27002")`  
        - `rs.add("m103:27003")`      
    - Check replica topology again with `rs.isMaster()`            

    - Sample config file (.conf) for a node of replica set

            storage:
              dbPath: "/data/db"
            systemLog:
              path: "/data/log/mongod.log"
              destination: "file"
            replication:
              replSetName: m103
            net:
              bindIp : "127.0.0.1,192.168.103.100"
              port: 27001
            security:
              keyFile: "/data/keyfile"
            processManagement:
              fork: true

- **Setting Up a Sharded Cluster**
    - **INFORMATIONS** 
        - The way distribute data in MongoDB is called Sharding.
        - Dataset divided into as many shards as you want.
        - Shards make up Sharded Cluster
        - Each shard in Sharded Cluster is a replica set
        - Mongos is acting like a router that accepts request from client and redirect.
        - Mongos uses metadata about which data contained on each shard.
        - These metadata stored on Config Servers. (**CSRS**: Config Server Replica Set)
        - Mongos use config servers very often.
        - These config files uses a _**keyfile**_ but bigger project _**x509**_ 
        - Config Servers are replica sets too.
    - Setting Up
        - Create configuration files (.conf) for every sharding replica set member (node).
        - These config files similar to standard replica sets members but there are some additions. See below. (see YAML file syntax for config files)
            - Difference is at the first rows:
            
                    sharding:
                      clusterRole: configsvr
        - With these config files start mongo daemons (for example for 1st one)
            - `mongod -f csrc_1.conf` (and other configs at odd numbers)                 - After starting all config server replica members connect to first one (or anyone you choose)
                - `mongo --port <target members port>` 
        - And initiate CSRS (Config servers replica set) 
            - `rs.initiate()`        
        - Now create super user as we did before on standard replica sets
            
                use admin
                db.createUser({
                  user: "m103-admin",
                  pwd: "m103-pass",
                  roles: [
                    {role: "root", db: "admin"}
                  ]
                })

        - Just after authenticate as this super user with `db.auth("m103-admin", "m103-pass")`                 
        - Add 2nd and 3rd nodes into CSRS
            
                rs.add("192.168.103.100:26002")
                rs.add("192.168.103.100:26003")
                rs.add("<members ip number:<members port number>")
        - Create mongos configuration file (.conf)
        
                sharding:
                  configDB: m103-csrs/192.168.103.100:26001,192.168.103.100:26002,192.168.103.100:26003
                security:
                  keyFile: /var/mongodb/pki/m103-keyfile
                net:
                  bindIp: localhost,192.168.103.100
                  port: 26000
                systemLog:
                  destination: file
                  path: /var/mongodb/db/mongos.log
                  logAppend: true
                processManagement:
                  fork: true

        - Connect to mongos with this connection commend (attention to port number)
         `mongo --port 26000 --username m103-admin --password m103-pass --authenticationDatabase admin`
         
         - You can check sharding status with `sh.status()`
         
- **Adding Replica Sets to Sharding Cluster**
    
    - First update replica sets numbers (nodes) config files
        - Update with and add this part at the beginnings of each file
        
                sharding:
                  clusterRole: shardsvr
                storage:
                  dbPath: /var/mongodb/db/node1
                  wiredTiger:
                    engineConfig:
                      cacheSizeGB: .1 
        - If they are running you can shutdown with  
                
                use admin
                db.shutdownServer()
                // and restart with 
                mongod -f mongod-1.conf 
                
        - P.S. : You can connect to any member of replica set with `mongo --port <port number> -u "m103-admin" -p "m103-pass" --authenticationDatabase "admin"`        
        - To add these clusters to mongos 
            - Connect to mongos
            
                    mongo --port <port number> --username m103-admin --password m103-pass --authenticationDatabase admin
            
            `sh.addShard("<replica set name>/primary members ip: primary members port number")`

        - Now you have one mongos, one SCRS (Sharding Cluster Replica Set) and one Cluster (Replica Set)

- **Sharding the Collection**
    
    - Connect to mongos
    - Enable sharding first to any database
    
            `sh.enableSharding("<database name>")`
    - Choose proper **shard key** from a collection
    - First create an Index for this key
    
               db.<database name>.createIndex({"<shard_key>": 1})
    - Now shard the collection with the command:
    
            db.adminCommand( { shardCollection: "<database name>.<collectionname>", key: { <shard_key>: 1 } } ) 

    - thats all for now.
