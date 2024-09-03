# MongoDB Sharded Cluster with Heavy-Read and Heavy-Write Shards

This repository provides instructions for setting up a MongoDB sharded cluster on CentOS with two shards: one optimized for heavy-write operations and the other for heavy-read operations. The setup also includes a `mongos` router to simplify connections and provide scalability and high availability.

## Prerequisites

- **CentOS** installed on all servers
- **MongoDB Community Edition** installed on all servers
- **Basic understanding of MongoDB sharding** and **replication**
- **SSH access** to all servers

## Architecture Overview

### IP Addresses

- **Write-Heavy Shard (Shard 1):**
  - Primary: `192.168.0.10`
  - Secondary 1: `192.168.0.11`
  - Secondary 2: `192.168.0.12`
  
- **Read-Heavy Shard (Shard 2):**
  - Primary: `192.168.0.20`
  - Secondary 1: `192.168.0.21`
  - Secondary 2: `192.168.0.22`
  
- **Config Servers:**
  - Config Server 1: `192.168.0.30`
  - Config Server 2: `192.168.0.31`
  - Config Server 3: `192.168.0.32`

- **Mongos Router:**
  - Mongos: `192.168.0.40`

### Components

- **Shards:** Two replica sets optimized for heavy-read and heavy-write operations.
- **Config Servers:** Store metadata and configuration settings for the sharded cluster.
- **Mongos Router:** Acts as the entry point for client applications, routing queries to the appropriate shard.

## Step-by-Step Setup

### 1. Install MongoDB on All Servers

```bash
sudo yum install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 2. Configure Config Servers

On each config server (`192.168.0.30`, `192.168.0.31`, `192.168.0.32`):

Edit `/etc/mongod.conf`:

```yaml
sharding:
  clusterRole: configsvr
replication:
  replSetName: rsConfig
net:
  bindIp: 0.0.0.0
  port: 27017
```

Start MongoDB and initialize the replica set:

```bash
mongo --port 27017
rs.initiate({
  _id: "rsConfig",
  configsvr: true,
  members: [
    { _id: 0, host: "192.168.0.30:27017" },
    { _id: 1, host: "192.168.0.31:27017" },
    { _id: 2, host: "192.168.0.32:27017" }
  ]
})
```

### 3. Configure Write-Heavy Shard (Shard 1)

On each server in Shard 1 (`192.168.0.10`, `192.168.0.11`, `192.168.0.12`):

Edit `/etc/mongod.conf`:

```yaml
sharding:
  clusterRole: shardsvr
replication:
  replSetName: rsWriteHeavy
net:
  bindIp: 0.0.0.0
  port: 27018
```

Start MongoDB and initialize the replica set on the primary (`192.168.0.10`):

```bash
mongo --port 27018
rs.initiate({
  _id: "rsWriteHeavy",
  members: [
    { _id: 0, host: "192.168.0.10:27018" },
    { _id: 1, host: "192.168.0.11:27018" },
    { _id: 2, host: "192.168.0.12:27018" }
  ]
})
```

### 4. Configure Read-Heavy Shard (Shard 2)

On each server in Shard 2 (`192.168.0.20`, `192.168.0.21`, `192.168.0.22`):

Edit `/etc/mongod.conf`:

```yaml
sharding:
  clusterRole: shardsvr
replication:
  replSetName: rsReadHeavy
net:
  bindIp: 0.0.0.0
  port: 27019
```

Start MongoDB and initialize the replica set on the primary (`192.168.0.20`):

```bash
mongo --port 27019
rs.initiate({
  _id: "rsReadHeavy",
  members: [
    { _id: 0, host: "192.168.0.20:27019" },
    { _id: 1, host: "192.168.0.21:27019" },
    { _id: 2, host: "192.168.0.22:27019" }
  ]
})
```

### 5. Set Up the Mongos Router

On the Mongos server (`192.168.0.40`):

Edit `/etc/mongos.conf`:

```yaml
sharding:
  configDB: rsConfig/192.168.0.30:27017,192.168.0.31:27017,192.168.0.32:27017
net:
  bindIp: 0.0.0.0
  port: 27020
```

Start the `mongos` service:

```bash
sudo systemctl start mongos
sudo systemctl enable mongos
```

### 6. Add the Shards to the Cluster

Connect to the `mongos` router (`192.168.0.40`):

```bash
mongo --port 27020
```

Add both shards to the cluster:

```js
sh.addShard("rsWriteHeavy/192.168.0.10:27018")
sh.addShard("rsReadHeavy/192.168.0.20:27019")
```

### 7. Connection Strings

#### **Write Operations**

Use this connection string for write-heavy operations:

```plaintext
mongodb://192.168.0.10:27018,192.168.0.11:27018,192.168.0.12:27018/?replicaSet=rsWriteHeavy
```

#### **Read Operations**

Use this connection string for read-heavy operations, preferring secondary nodes:

```plaintext
mongodb://192.168.0.20:27019,192.168.0.21:27019,192.168.0.22:27019/?replicaSet=rsReadHeavy&readPreference=secondaryPreferred
```

#### **Unified Connection via Mongos Router**

Alternatively, use the `mongos` router as a single entry point:

```plaintext
mongodb://192.168.0.40:27020/?readPreference=secondaryPreferred
```

### 8. Router vs. Direct Connection: Which to Use?

#### **Using the Mongos Router**

**Advantages:**
- **Simplified Architecture:** A single connection point simplifies the application logic. The `mongos` router handles the distribution of queries to the appropriate shards, making it easier to manage a complex sharded cluster.
- **Scalability:** As your cluster grows, you can add more `mongos` instances to handle increased traffic. This is particularly useful in large-scale environments.
- **Automatic Load Balancing:** `mongos` automatically routes queries based on the shard key, which helps balance the load across shards without manual intervention.
- **High Availability:** If one shard becomes unavailable, `mongos` can route operations to other available shards, improving fault tolerance.

**Best For:** 
- Large-scale deployments
- Applications requiring high availability and automated failover
- Scenarios where managing multiple connection strings is impractical

#### **Using Direct Connections**

**Advantages:**
- **Optimized Performance:** Directly connecting to the specific shards allows for more fine-grained control, which can reduce latency by bypassing the `mongos` layer.
- **Custom Routing:** You can manually control where reads and writes go, allowing for more optimized resource usage in specialized workloads.

**Disadvantages:**
- **Increased Complexity:** Managing separate connections for read-heavy and write-heavy operations adds complexity to your application. This can be difficult to scale and maintain as your cluster grows.
- **Manual Failover Management:** In the event of a failure, your application needs to handle failover manually, which can add to the complexity.

**Best For:**
- Small or specialized applications where performance is critical
- Environments where you need to optimize specific workloads manually

### 9. Monitoring and Maintenance

- **Monitoring:** Use tools like `mongostat` and `mongotop` to monitor the performance of your MongoDB cluster.
- **Backups:** Regularly back up your data from both shards and config servers.

### 10. Scaling Considerations

- **Scaling Shards:** Add more shards as your data grows. MongoDB automatically balances data across shards.
- **Scaling Mongos:** Deploy multiple `mongos` instances with a load balancer to handle increased traffic.

## Conclusion

This setup provides a robust, scalable solution for handling read-heavy and write-heavy

 workloads in MongoDB. By using a sharded architecture with dedicated shards for different types of operations, you can optimize performance and ensure high availability.

For most scenarios, using the `mongos` router as a single entry point is recommended due to its simplicity and scalability. However, in cases where maximum performance and fine-grained control are required, direct connections to specific shards may be beneficial.

Feel free to fork this repository and contribute improvements or suggestions!
