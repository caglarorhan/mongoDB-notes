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
**Replication**
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
    - Start/run any odd number of mongod nodes. Odd is required to election (voting) for Primer.
    - Connect first node you create with `mongo --port <given port on conf file>`    

**Sample Config File for A Node of Replica Set**

    storage:
      dbPath: "/data/db"
    systemLog:
      path: "/data/log/mongod.log"
      destination: "file"
    replication:
      replSetName: M103
    net:
      bindIp : "127.0.0.1,192.168.103.100"
      port: 27000
    tls:
      mode: "requireTLS"
      certificateKeyFile: "/etc/tls/tls.pem"
      CAFile: "/etc/tls/TLSCA.pem"
    security:
      keyFile: "/data/keyfile"
    processManagement:
      fork: true
