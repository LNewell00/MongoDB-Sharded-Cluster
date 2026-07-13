# MongoDB Sharded Cluster Deployment & Testing Guide

This guide covers setting up a MongoDB Sharded Cluster on OpenSUSE Tumbleweed using Docker. It provides three deployment options depending on your availability: **Multi-Machine (4 VMs)**, **Single-VM (Docker CLI)**, and **Single-VM (Docker Compose)**. It also includes CRUD testing instructions to verify your sharded network.

---

## 1. Architectural Background

*   **Config Server:** Stores cluster metadata (mapping of chunks to shards, cluster settings, and routing info). It does *not* store application data and must always be configured as a replica set. The cluster cannot function or route requests without a healthy config server replica set.
*   **Shard Server:** Stores the actual user/application data. Each shard handles a distinct partition (partitioned by a shard key) of the overall dataset. Shards are typically configured as replica sets to ensure high availability and data redundancy.
*   **Router (`mongos`):** The interface/entry point for applications. Applications connect to `mongos` exactly like a standard MongoDB instance. The `mongos` router queries the config servers to determine where specific documents reside and routes client operations to the appropriate shards.

---

## 2. Infrastructure Prerequisites

*   **Operating System:** OpenSUSE Tumbleweed
*   **Base Specifications (Per Machine/VM):** Minimum 4GB RAM | 1 CPU | Static IP | SSH enabled
*   **VirtualBox Network Configuration:** Use a *NAT Network* or *Bridged Adapter* to ensure cross-VM communication.

### Installing and Enabling Docker (Run on all nodes)
```bash
sudo zypper install docker
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 3. Deployment Option 1: Multi-Machine Setup (4 VMs)

This architecture distributes the database layers across four distinct virtual or physical servers to replicate a production-grade infrastructure.

### Machine 2, 3, and 4 — Data Nodes (Config + Shard)
Run these commands on **each** of the three data machines to spin up one Config server and one Shard server container per node.

```bash
# Create persistent volume and run the Config Server container
docker volume create config-data
docker run -d --restart unless-stopped --name config-node -p 27019:27019 \
  -v config-data:/data/db   mongodb/mongodb-enterprise-server:latest \
  mongod --configsvr --replSet configRS --port 27019

# Create persistent volume and run the Shard Server container
docker volume create shard-data
docker run -d --restart unless-stopped --name shard-node -p 27018:27018 \
  -v shard-data:/data/db   mongodb/mongodb-enterprise-server:latest   mongod --shardsvr --replSet shardRS --port 27018
```

### Initiate Config Replica Set (Run ONLY on Machine 2)
```bash
docker exec -it config-node mongosh --port 27019

rs.initiate({
  _id: "configRS",
  configsvr: true,
  members: [
    { _id: 0, host: "<Machine2_IP>:27019" },
    { _id: 1, host: "<Machine3_IP>:27019" },
    { _id: 2, host: "<Machine4_IP>:27019" }
  ]
})
```

### Initiate Shard Replica Set (Run ONLY on Machine 2)
```bash
docker exec -it shard-node mongosh --port 27018

rs.initiate({
  _id: "shardRS",
  members: [
    { _id: 0, host: "<Machine2_IP>:27018" },
    { _id: 1, host: "<Machine3_IP>:27018" },
    { _id: 2, host: "<Machine4_IP>:27018" }
  ]
})
```

### Machine 1 — Create Router (`mongos`)
Run this command on your dedicated application router machine to point it to the config server cluster metadata.
```bash
docker run -d --name mongos -p 27017:27017 \
  mongodb/mongodb-enterprise-server:latest mongos \
  --configdb configRS/<Machine2_IP>:27019,<Machine3_IP>:27019,<Machine4_IP>:27019 \
  --bind_ip_all
```

### Connect to Router and Link Shards (Run on Machine 1)
```bash
docker exec -it mongos mongosh --port 27017

sh.addShard("shardRS/<Machine2_IP>:27018,<Machine3_IP>:27018,<Machine4_IP>:27018")
```
*(Remember to replace `<MachineX_IP>` placeholders with your static network IPs obtained from `ip a`.)*

---

## 4. Deployment Option 2: Single-VM Setup (Docker CLI)

This configuration consolidates all components onto a single machine or virtual instance by utilizing an isolated Docker network bridge for container communications.

### Step 2.1: Provision the Network Bridge
```bash
docker network create mongodb-network
```

### Step 2.2: Provision Config Servers
```bash
docker volume create config1-data
docker run -d --name config1 --network mongodb-network -p 27019:27019 \
  -v config1-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --configsvr --replSet configRS --port 27019

docker volume create config2-data
docker run -d --name config2 --network mongodb-network -p 27020:27019 \
  -v config2-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --configsvr --replSet configRS --port 27019

docker volume create config3-data
docker run -d --name config3 --network mongodb-network -p 27021:27019 \
  -v config3-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --configsvr --replSet configRS --port 27019
```

### Step 2.3: Provision Shard Servers
```bash
docker volume create shard1a-data
docker run -d --name shard1a --network mongodb-network -p 27022:27018 \
  -v shard1a-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --shardsvr --replSet shardRS1 --port 27018

docker volume create shard1b-data
docker run -d --name shard1b --network mongodb-network -p 27023:27018 \
  -v shard1b-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --shardsvr --replSet shardRS1 --port 27018

docker volume create shard1c-data
docker run -d --name shard1c --network mongodb-network -p 27024:27018 \
  -v shard1c-data:/data/db \
  mongodb/mongodb-enterprise-server:latest mongod --shardsvr --replSet shardRS1 --port 27018
```

### Step 2.4: Sequentially Initialize Replica Sets
> ⚠️ **Crucial Note:** You must explicitly execute into a config container to initialize `configRS`, and a shard container to initialize `shardRS1`. Cross-contamination will throw configuration errors.

```bash
# Connect and initialize Config Server Replica Set
docker exec -it config1 mongosh --port 27019

rs.initiate({
  _id: "configRS",
  configsvr: true,
  members: [
    { _id: 0, host: "config1:27019" },
    { _id: 1, host: "config2:27019" },
    { _id: 2, host: "config3:27019" }
  ]
})
exit

# Connect and initialize Shard Server Replica Set
docker exec -it shard1a mongosh --port 27018

rs.initiate({
  _id: "shardRS1",
  members: [
    { _id: 0, host: "shard1a:27018" },
    { _id: 1, host: "shard1b:27018" },
    { _id: 2, host: "shard1c:27018" }
  ]
})
exit
```

### Step 2.5: Provision Router and Register Shards
```bash
# Instantiate the Router
docker run -d --name mongos --network mongodb-network -p 27017:27017 \
  mongodb/mongodb-enterprise-server:latest mongos \
  --configdb configRS/config1:27019,config2:27019,config3:27019 --bind_ip_all

# Connect to router and introduce the shard layer
docker exec -it mongos mongosh --port 27017

sh.addShard("shardRS1/shard1a:27018,shard1b:27018,shard1c:27018")
```

---

## 5. Deployment Option 3: Single-VM Setup (Docker Compose)

This configuration automates the creation of all containers, networking volumes, and routing layers in a single declarative configuration file.

> 💡 **Operational Tip:** To ensure high structural reliability, automation scripts have been removed from the `mongos` deployment logic inside the compose file. This mitigates start-up race conditions (where `mongos` throws crashes because dependencies haven't passed health checks yet). Spin up the environment via compose, then execute **Step 2.4 & 2.5** above to seamlessly bind your active cluster components together.

```yaml
version: "3.9"

services:
  config1:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: config1
    command: ["mongod", "--configsvr", "--replSet", "configRS", "--port", "27019"]
    ports: ["27019:27019"]
    networks: ["mongo_net"]
    volumes: ["config1-data:/data/db"]

  config2:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: config2
    command: ["mongod", "--configsvr", "--replSet", "configRS", "--port", "27019"]
    ports: ["27020:27019"]
    networks: ["mongo_net"]
    volumes: ["config2-data:/data/db"]

  config3:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: config3
    command: ["mongod", "--configsvr", "--replSet", "configRS", "--port", "27019"]
    ports: ["27021:27019"]
    networks: ["mongo_net"]
    volumes: ["config3-data:/data/db"]

  shard1a:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: shard1a
    command: ["mongod", "--shardsvr", "--replSet", "shard1RS", "--port", "27018"]
    ports: ["27022:27018"]
    networks: ["mongo_net"]
    volumes: ["shard1a-data:/data/db"]

  shard1b:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: shard1b
    command: ["mongod", "--shardsvr", "--replSet", "shard1RS", "--port", "27018"]
    ports: ["27023:27018"]
    networks: ["mongo_net"]
    volumes: ["shard1b-data:/data/db"]

  shard1c:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: shard1c
    command: ["mongod", "--shardsvr", "--replSet", "shard1RS", "--port", "27018"]
    ports: ["27024:27018"]
    networks: ["mongo_net"]
    volumes: ["shard1c-data:/data/db"]

  mongos:
    image: mongodb/mongodb-enterprise-server:latest
    container_name: mongos
    command: ["mongos", "--configdb", "configRS/config1:27019,config2:27019,config3:27019", "--port", "27017", "--bind_ip_all"]
    ports: ["27017:27017"]
    networks: ["mongo_net"]
    depends_on:
      - config1
      - config2
      - config3
      - shard1a
      - shard1b
      - shard1c

networks:
  mongo_net:
    driver: bridge

volumes:
  config1-data:
  config2-data:
  config3-data:
  shard1a-data:
  shard1b-data:
  shard1c-data:
```

Command to start infrastructure:
```bash
docker compose up -d
```

---

## 6. Cluster Verification (JSON CRUD Operations)

Once your sharded cluster is initialized using any of the three techniques outlined above, log into the active cluster router container to test JSON lifecycle processing.

```bash
docker exec -it mongos mongosh --port 27017
```

Execute the following database queries sequentially inside your open database shell interface (`mongosh`):

### Step 6.1: Initialize Database and Sharding Strategy
```javascript
// Switch to our target database environment
use myNewDB

// Enable clustering properties on the database container
sh.enableSharding("myNewDB")

// Sharding partitions data on a chosen index key. We will generate a hashed _id key index.
db.myCollection.createIndex({ _id: "hashed" })

// Formally partition the collection dynamically across the shard topology via the key index
sh.shardCollection("myNewDB.myCollection", { _id: "hashed" })
```

### Step 6.2: CREATE (Insert JSON Document)
```javascript
db.myCollection.insertOne({
  "title": "MongoDB Cluster Test",
  "type": "Research Project",
  "status": "Active",
  "details": {
    "verified": true,
    "framework": "Docker containers"
  },
  "version": 1
})
```
*Expected Return Value: `{ acknowledged: true, insertedId: ObjectId(...) }`*

### Step 6.3: READ (Retrieve JSON Document)
```javascript
db.myCollection.find({ title: "MongoDB Cluster Test" }).pretty()
```

### Step 6.4: UPDATE (Modify JSON Document)
```javascript
db.myCollection.updateOne(
  { title: "MongoDB Cluster Test" },
  { 
    $set: { "status": "Successfully Sharded" },
    $inc: { "version": 1 } 
  }
)

// Re-read back the data to ensure document update parameters matched successfully
db.myCollection.find({ title: "MongoDB Cluster Test" })
```

### Step 6.5: DELETE (Remove JSON Document)
```javascript
db.myCollection.deleteOne({ title: "MongoDB Cluster Test" })

// Re-read query to prove data was cleanly extracted out of active storage arrays (returns nothing)
db.myCollection.find({ title: "MongoDB Cluster Test" })
```
