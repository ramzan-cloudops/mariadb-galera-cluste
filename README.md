# MariaDB Galera Cluster Documentation


# 1. Introduction

Modern applications require databases that can remain available even when individual servers fail. A traditional single-database architecture can create a single point of failure, while asynchronous replication can introduce replica lag between the primary server and its replicas.

**MariaDB Galera Cluster** provides a high-availability, multi-primary database architecture based on Galera Cluster technology. It uses virtually synchronous, certification-based replication to keep cluster members coordinated and allows multiple nodes to participate in database operations.

A typical deployment consists of multiple MariaDB servers connected through the Galera replication layer.

### High-Level Architecture

<!-- IMAGE PLACEHOLDER: MariaDB Galera Cluster Architecture -->

```text
                         Application
                              |
                              v
                    +-------------------+
                    | Load Balancer /   |
                    | MariaDB MaxScale  |
                    +---------+---------+
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
        +----------+     +----------+     +----------+
        | Node 1   |     | Node 2   |     | Node 3   |
        | MariaDB  |     | MariaDB  |     | MariaDB  |
        | Galera   |     | Galera   |     | Galera   |
        +----+-----+     +----+-----+     +----+-----+
             |                |                |
             +----------------+----------------+
                      Galera Replication
```

The goal of this architecture is to provide:

* High availability
* Data consistency
* Multi-primary operation
* Automatic node membership and recovery
* Reduced dependency on a single database server
* Read scalability through appropriate traffic routing

MariaDB continues to include Galera Cluster in its Community Server distribution and maintains it as a long-term clustering technology.

---

# 2. What is MariaDB Galera Cluster?

**MariaDB Galera Cluster** is a database clustering solution that connects multiple MariaDB servers into a coordinated cluster.

Galera provides:

* Virtually synchronous replication
* Multi-primary database operation
* Group communication
* Transaction certification
* Automatic membership management
* State synchronization
* Quorum-based cluster operation

Unlike a traditional primary/replica architecture, Galera does not require one database server to be the permanent source of truth for all writes.

Instead, multiple nodes can participate in the cluster.

```text
Traditional Replication

              Primary
                 |
          +------+------+
          |             |
          v             v
       Replica 1     Replica 2


Galera Cluster

          +-------------+
          |   Node 1    |
          +-------------+
             ↕       ↕
          +-------------+
          |   Node 2    |
          +-------------+
             ↕       ↕
          +-------------+
          |   Node 3    |
          +-------------+
```

This architecture makes Galera particularly useful for applications where database availability and consistency are important.

---

# 3. Why Use MariaDB Galera Cluster?

A single MariaDB server creates a potential single point of failure:

```text
Application
     |
     v
+------------+
| MariaDB    |
| Server     |
+------------+
     |
   FAILURE
     |
     X
```

If the database server becomes unavailable, the application may also become unavailable.

A Galera cluster distributes the database across multiple nodes:

```text
                 Application
                      |
                      v
               Cluster Endpoint
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       Node 1      Node 2      Node 3
          \           |           /
           \          |          /
            +---------+---------+
                 Galera
               Replication
```

If one node fails, the remaining nodes can continue operating as long as the cluster retains a valid Primary Component.

This provides a foundation for high availability without relying on a single database server.

---

# 4. Core Architecture

A MariaDB Galera Cluster consists of several important components.

## 4.1 MariaDB Server

Each cluster node runs a MariaDB Server instance.

The MariaDB server handles:

* SQL queries
* Transactions
* Storage
* Authentication
* Database operations

---

## 4.2 WSREP API

Galera integrates with MariaDB through the **Write Set Replication (WSREP)** interface.

The general architecture is:

```text
+----------------------------------+
|          Application             |
+----------------+-----------------+
                 |
                 v
+----------------------------------+
|          MariaDB Server          |
+----------------+-----------------+
                 |
                 v
+----------------------------------+
|             WSREP                |
|      Write Set Replication      |
+----------------+-----------------+
                 |
                 v
+----------------------------------+
|       Galera Replication        |
|                                  |
|  Group Communication             |
|  Write-Set Replication           |
|  Certification                   |
|  Global Ordering                 |
+----------------------------------+
```

The WSREP/Galera layer coordinates replication between the database nodes.

---

## 4.3 Galera Provider

The Galera provider implements the replication functionality used by the MariaDB server.

A modern MariaDB installation commonly uses the `galera-4` provider.

The provider is responsible for important cluster functions such as:

* Replication
* Group communication
* Transaction ordering
* Certification
* Cluster membership
* State transfer

---

## 4.4 Cluster Nodes

Each participating MariaDB server is called a **cluster node**.

For example:

```text
Cluster Name: production_mariadb

+----------+      +----------+      +----------+
| Node 1   |      | Node 2   |      | Node 3   |
| DB-01    |      | DB-02    |      | DB-03    |
+----------+      +----------+      +----------+
      \               |               /
       \              |              /
        +-------------+-------------+
              Galera Cluster
```

Each node maintains its own local database storage while Galera coordinates changes between nodes.

---

# 5. Key Features

MariaDB Galera Cluster provides several important capabilities.

| Feature                               | Description                                                                                  |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Multi-Primary**                     | Multiple cluster nodes can accept database operations.                                       |
| **Virtually Synchronous Replication** | Transactions are coordinated across active cluster members.                                  |
| **Certification-Based Replication**   | Conflicting transactions can be detected during certification.                               |
| **Automatic Membership**              | Nodes can join and leave the cluster dynamically.                                            |
| **IST**                               | Transfers missing transactions to a recovering node when possible.                           |
| **SST**                               | Performs a full state transfer when incremental recovery is not possible.                    |
| **Quorum**                            | Prevents isolated groups from independently operating as the Primary Component.              |
| **Flow Control**                      | Helps prevent faster nodes from overwhelming slower nodes.                                   |
| **High Availability**                 | Remaining nodes can continue serving workloads after a node failure if quorum is maintained. |
| **Read Scalability**                  | Reads can be distributed across suitable nodes using an appropriate routing layer.           |

MariaDB's current Galera documentation describes the technology around virtually synchronous replication, cluster membership, state transfers and high availability.

---

# 6. Multi-Primary Architecture

One of the defining characteristics of Galera is its **multi-primary architecture**.

In a traditional primary/replica setup:

```text
             Primary
                |
       +--------+--------+
       |                 |
       v                 v
    Replica 1         Replica 2
```

Writes normally go to the primary.

With Galera:

```text
        +----------+
        |  Node 1  |
        +----------+
          ↕      ↕
        +----------+
        |  Node 2  |
        +----------+
          ↕      ↕
        +----------+
        |  Node 3  |
        +----------+
```

Each node can potentially accept both reads and writes.

For example:

```text
Application A ---> Node 1 ---> Write
Application B ---> Node 2 ---> Write
Application C ---> Node 3 ---> Write
```

Galera replicates the resulting write-sets between the cluster members.

### Important Consideration

Multi-primary does **not** mean that adding more nodes provides unlimited write scalability.

Concurrent writes affecting the same data can conflict during certification. Applications and database designs should therefore account for transaction conflicts and appropriate retry behavior.

---

# 7. Virtually Synchronous Replication

Galera uses a **virtually synchronous** replication model.

A simplified transaction flow is:

```text
             Client
                |
                v
          Execute Transaction
                |
                v
          Create Write-Set
                |
                v
       Broadcast to Cluster
                |
                v
           Certification
             /       \
            /         \
         PASS         FAIL
          |             |
          v             v
       Commit        Rollback
```

The transaction is coordinated through the Galera replication layer before the commit is finalized.

This differs from traditional asynchronous replication, where a primary can commit a transaction before replicas have received or applied the change.

The virtually synchronous model helps reduce the traditional replica-lag problem and provides a strong consistency model for active cluster members.

---

# 8. Certification-Based Replication

Galera uses **write-set certification** to detect conflicting transactions.

Consider two transactions:

```text
Node 1:
UPDATE accounts
SET balance = balance - 100
WHERE id = 10;


Node 2:
UPDATE accounts
SET balance = balance - 200
WHERE id = 10;
```

Both transactions may execute locally.

During certification, Galera determines whether their write-sets conflict.

```text
Transaction
     |
     v
Write-Set
     |
     v
Cluster Certification
     |
  +--+--+
  |     |
Pass   Conflict
  |       |
  v       v
Commit  Rollback
```

If a transaction fails certification, the application may receive a transaction conflict/deadlock error and should handle the error appropriately.

This certification mechanism is a core part of Galera's multi-primary replication architecture.

---

# 9. Quorum and Split-Brain Prevention

Galera uses **quorum** to determine whether a group of nodes can continue operating as the Primary Component.

Consider a three-node cluster:

```text
+--------+     +--------+     +--------+
| Node 1 |     | Node 2 |     | Node 3 |
+--------+     +--------+     +--------+
```

If Node 3 fails:

```text
+--------+     +--------+
| Node 1 |-----| Node 2 |
+--------+     +--------+

Node 3 = FAILED
```

Two of the three nodes remain connected, so the majority can continue operating.

### Network Partition

Suppose the network separates the nodes:

```text
        Network Partition
              X
              |
      +-------+-------+
      |               |
   Node 1           Node 2
                     Node 3
```

Galera's quorum mechanism prevents both sides from independently becoming valid Primary Components.

This protects against **split-brain**, where two isolated groups could otherwise accept conflicting changes.

### Important Planning Rule

A three-node cluster is a common production topology because it can tolerate the failure of one voting member while maintaining quorum.

Galera also supports deployment variants involving **Galera Arbitrator (`garbd`)**, which can provide an additional voting member without storing the database itself. Therefore, “three database servers are mandatory” is too simplistic; the correct design depends on the required topology and failure model.

---

# 10. Node States

Galera nodes move through different states during startup, synchronization and normal operation.

Common states include:

| State         | Description                                                         |
| ------------- | ------------------------------------------------------------------- |
| `Joining`     | The node is joining the cluster.                                    |
| `Joined`      | The node has joined but may still be completing synchronization.    |
| `Synced`      | The node is synchronized and ready for normal cluster operation.    |
| `Donor`       | The node is providing state to another node.                        |
| `Desynced`    | The node is temporarily outside normal synchronization flow.        |
| `Non-Primary` | The node is not currently part of an operational Primary Component. |

The most important healthy state for a serving node is:

```text
Synced
```

This can be checked with:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

Expected output:

```text
wsrep_local_state_comment    Synced
```

---

# 11. IST and SST

When a node joins or rejoins the cluster, it needs to obtain any database state that it is missing.

Galera provides two major state-transfer mechanisms:

```text
                Joining Node
                     |
             +-------+-------+
             |               |
          IST possible?       |
             |               |
            YES              NO
             |               |
             v               v
            IST              SST
             |               |
             v               v
       Missing changes    Full state
```

## 11.1 Incremental State Transfer — IST

**IST (Incremental State Transfer)** transfers only the write-sets that the joining node missed.

Example:

```text
Node 1:
Transactions 1 - 1000

Node 2:
Transactions 1 - 950

Missing:
951 - 1000
```

IST can transfer only:

```text
951 - 1000
```

This is significantly more efficient than transferring the entire database.

---

## 11.2 State Snapshot Transfer — SST

If the required write-sets are no longer available for IST, the node may require an **SST (State Snapshot Transfer)**.

Conceptually:

```text
Donor Node
    |
    | Full Database State
    v
Joining Node
```

SST is more resource-intensive because it transfers a much larger amount of data.

Therefore:

```text
IST = Preferred when possible
SST = Full-state recovery when required
```

MariaDB documentation identifies IST and SST as the mechanisms used to synchronize joining or recovering nodes. 


# 12. Flow Control

**Flow Control** is a mechanism used by Galera to prevent faster nodes from overwhelming slower nodes.

Consider a three-node cluster:

```text
Node 1  ────────>  Fast
Node 2  ────────>  Fast
Node 3  ────────>  Slow
```

If Node 3 cannot apply replicated transactions quickly enough, its receive queue can grow:

```text
              Write-Sets
                   |
                   v
             +-----------+
             |  Node 3   |
             +-----------+
                   |
              Receive Queue
                   |
                Growing
```

Galera can activate Flow Control to temporarily slow down replication while the slower node catches up.

```text
Fast Nodes
    |
    | Replication
    v
Slow Node
    |
    | Queue becomes high
    v
Flow Control
    |
    v
Replication temporarily slowed
```

### Why is Flow Control Important?

Without Flow Control, a slow node could continuously fall further behind and consume increasing amounts of memory.

Flow Control helps keep cluster members within a manageable synchronization range.

### Common Causes of Flow Control

High Flow Control can indicate:

* Slow disk I/O
* Insufficient CPU
* Memory pressure
* Network latency
* Large transactions
* High database workload
* Slow replication threads
* An overloaded cluster node

### Monitoring Flow Control

Use:

```sql
SHOW STATUS LIKE 'wsrep_flow_control%';
```

Also inspect the receive queue:

```sql
SHOW STATUS LIKE 'wsrep_local_recv_queue%';
```

Important metrics include:

* `wsrep_flow_control_paused`
* `wsrep_flow_control_sent`
* `wsrep_flow_control_recv`
* `wsrep_local_recv_queue`
* `wsrep_local_recv_queue_avg`

A consistently high Flow Control rate should be investigated rather than simply disabled. MariaDB documents Flow Control as an important mechanism for keeping nodes synchronized when some nodes cannot process replicated writes as quickly as others.

<!-- IMAGE PLACEHOLDER: Galera Flow Control Diagram -->

---

# 13. Prerequisites

Before deploying MariaDB Galera Cluster, all participating servers should be properly prepared.

A basic production topology can contain:

```text
                    Galera Cluster

          +-----------------------------+
          |                             |
     +---------+    +---------+    +---------+
     | Node 1  |    | Node 2  |    | Node 3  |
     | MariaDB |    | MariaDB |    | MariaDB |
     +---------+    +---------+    +---------+
          \              |              /
           \             |             /
            +------------+------------+
                 Cluster Network
```

## 13.1 Operating System

MariaDB Galera Cluster runs on Linux.

For a production deployment:

* Use a supported Linux distribution.
* Keep the operating system versions consistent where practical.
* Keep MariaDB versions compatible across cluster nodes.
* Keep Galera provider versions compatible.
* Synchronize system time using NTP or an equivalent time synchronization service.

Example:

```bash
timedatectl status
```

Check whether time synchronization is enabled:

```bash
timedatectl show-timesync --all
```

---

# 14. Infrastructure Requirements

A production cluster should be designed with failure tolerance in mind.

A common topology is:

```text
Availability Zone A
+-------------------+
| Node 1            |
+-------------------+

Availability Zone B
+-------------------+
| Node 2            |
+-------------------+

Availability Zone C
+-------------------+
| Node 3            |
+-------------------+
```

This reduces the risk of losing the entire cluster because of a single host, rack or availability-zone failure.

### Recommended Planning Areas

| Area         | Consideration                                         |
| ------------ | ----------------------------------------------------- |
| CPU          | Enough capacity for database workload and replication |
| RAM          | Enough memory for MariaDB buffers and workload        |
| Storage      | Fast and reliable storage                             |
| Network      | Stable and low-latency connection                     |
| Availability | Distribute nodes across failure domains               |
| OS           | Supported Linux distribution                          |
| MariaDB      | Compatible versions across nodes                      |
| Time         | Synchronized system clocks                            |
| Security     | Restricted cluster communication                      |

The exact CPU, RAM and storage requirements depend on the application's workload and should be determined through capacity testing rather than a fixed universal specification.

---

# 15. Network Requirements

Galera nodes must communicate with each other continuously.

A simplified network looks like:

```text
+---------+          +---------+
| Node 1  |<-------->| Node 2  |
+----+----+          +----+----+
     ^                    ^
     |                    |
     |                    |
     v                    v
+---------+<------------>+
| Node 3  |
+---------+
```

The following ports are commonly required:

|   Port | Protocol | Purpose                          |
| -----: | -------- | -------------------------------- |
| `3306` | TCP      | MariaDB client connections       |
| `4567` | TCP/UDP  | Galera replication               |
| `4568` | TCP      | Incremental State Transfer (IST) |
| `4444` | TCP      | State Snapshot Transfer (SST)    |

These ports should be allowed **only between trusted systems or networks** where possible.

Do not expose Galera replication ports directly to the public Internet.

---

# 16. Firewall Configuration

For an Ubuntu-based environment using UFW, an example configuration is:

```bash
sudo ufw allow 3306/tcp
sudo ufw allow 4567/tcp
sudo ufw allow 4567/udp
sudo ufw allow 4568/tcp
sudo ufw allow 4444/tcp
```

However, production firewall rules should restrict the source addresses.

For example, instead of allowing:

```text
0.0.0.0/0
```

the firewall should allow traffic only from the IP addresses or private networks used by the other Galera nodes.

Conceptually:

```text
Node 1 Private IP
       |
       | Allowed
       v
Node 2 Private IP

Node 3 Private IP
       |
       | Allowed
       v
Node 1 Private IP
```

---

# 17. Hostname and IP Planning

Before configuring the cluster, define the nodes clearly.

Example:

| Node   | Hostname        | IP Address  |
| ------ | --------------- | ----------- |
| Node 1 | `mariadb-node1` | `10.0.0.11` |
| Node 2 | `mariadb-node2` | `10.0.0.12` |
| Node 3 | `mariadb-node3` | `10.0.0.13` |

These are example addresses only.

Each node should have:

* A unique hostname
* A stable IP address
* Network connectivity to all other nodes
* Correct DNS or `/etc/hosts` resolution if hostnames are used

Example `/etc/hosts`:

```text
10.0.0.11 mariadb-node1
10.0.0.12 mariadb-node2
10.0.0.13 mariadb-node3
```

Verify connectivity:

```bash
ping mariadb-node2
```

and:

```bash
ping mariadb-node3
```

For production, also verify the required TCP ports rather than relying only on `ping`.

---

# 18. Software Requirements

Each node should have compatible versions of:

```text
MariaDB Server
MariaDB Client
Galera Provider
```

For modern MariaDB installations, the Galera provider package is commonly:

```text
galera-4
```

Example installation on Ubuntu:

```bash
sudo apt update
sudo apt install mariadb-server mariadb-client galera-4 -y
```

Verify MariaDB:

```bash
mariadb --version
```

Verify the installed Galera package:

```bash
dpkg -l | grep galera
```

The exact package names and installation procedure can vary by Linux distribution and MariaDB release.

---

# 19. Storage Engine Requirements

Galera is designed primarily around transactional workloads using **InnoDB**.

Set the default storage engine to:

```ini
default_storage_engine=InnoDB
```

Check the current value:

```sql
SHOW VARIABLES LIKE 'default_storage_engine';
```

Expected:

```text
default_storage_engine    InnoDB
```

Tables participating in replicated workloads should use a supported transactional storage engine.

---

# 20. Primary Keys

Primary keys are particularly important in Galera environments.

Tables should have an appropriate primary key:

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100)
);
```

Avoid creating important production tables without a primary key.

Primary keys help Galera identify rows efficiently during replication and certification.

Check a table:

```sql
SHOW CREATE TABLE users;
```

---

# 21. Galera Configuration

The main Galera configuration is normally placed in a MariaDB configuration directory.

On many Debian/Ubuntu installations, a common location is:

```text
/etc/mysql/mariadb.conf.d/galera.cnf
```

The exact path can vary depending on the operating system and package installation.

A basic configuration looks like:

```ini
[mysqld]

# Enable Galera
wsrep_on=ON

# Galera provider
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Cluster identity
wsrep_cluster_name="production_mariadb_cluster"

# Cluster members
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"

# Node identity
wsrep_node_name="mariadb-node1"
wsrep_node_address="10.0.0.11"

# Required replication settings
binlog_format=ROW
default_storage_engine=InnoDB
```

> **Note:** The Galera provider library path can differ between distributions and installations. Verify the actual installed library path on each server before using it.

---

# 22. Important Galera Configuration Parameters

## `wsrep_on`

```ini
wsrep_on=ON
```

Enables WSREP replication for the MariaDB server.

---

## `wsrep_provider`

```ini
wsrep_provider=/usr/lib/galera/libgalera_smm.so
```

Specifies the Galera provider library.

The path must match the provider installed on the system.

---

## `wsrep_cluster_name`

```ini
wsrep_cluster_name="production_mariadb_cluster"
```

Defines the name of the Galera cluster.

All nodes belonging to the same cluster should use the same cluster name.

---

## `wsrep_cluster_address`

```ini
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"
```

Defines the addresses used for cluster communication and node discovery.

It is good practice to provide the addresses of the available cluster members rather than relying on a single node.

---

## `wsrep_node_name`

```ini
wsrep_node_name="mariadb-node1"
```

Defines the unique name of the individual node.

Each node must have its own unique value.

For example:

```text
Node 1 → mariadb-node1
Node 2 → mariadb-node2
Node 3 → mariadb-node3
```

---

## `wsrep_node_address`

```ini
wsrep_node_address="10.0.0.11"
```

Specifies the network address used by the node.

Each server must use its own address.

---

## `binlog_format`

```ini
binlog_format=ROW
```

Galera requires row-based binary logging for normal operation.

Do not configure the cluster with statement-based replication for Galera workloads.

---

## `default_storage_engine`

```ini
default_storage_engine=InnoDB
```

Sets InnoDB as the default storage engine.

---

# 23. Configuration for Node 1

Example:

```ini
[mysqld]

wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="production_mariadb_cluster"
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"

wsrep_node_name="mariadb-node1"
wsrep_node_address="10.0.0.11"

binlog_format=ROW
default_storage_engine=InnoDB
```

---

# 24. Configuration for Node 2

Node 2 uses the same cluster configuration but has its own identity.

```ini
[mysqld]

wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="production_mariadb_cluster"
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"

wsrep_node_name="mariadb-node2"
wsrep_node_address="10.0.0.12"

binlog_format=ROW
default_storage_engine=InnoDB
```

---

# 25. Configuration for Node 3

```ini
[mysqld]

wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="production_mariadb_cluster"
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"

wsrep_node_name="mariadb-node3"
wsrep_node_address="10.0.0.13"

binlog_format=ROW
default_storage_engine=InnoDB
```

The main difference between the nodes is:

```text
wsrep_node_name
wsrep_node_address
```

The cluster name and cluster membership configuration remain consistent.

---

# 26. Verify the Configuration

Before starting the cluster, check the MariaDB configuration.

A useful command is:

```bash
mariadbd --verbose --help >/dev/null
```

Also verify the configuration file manually:

```bash
sudo cat /etc/mysql/mariadb.conf.d/galera.cnf
```

Check the service:

```bash
sudo systemctl status mariadb
```

If MariaDB is currently running before the first cluster bootstrap, stop it:

```bash
sudo systemctl stop mariadb
```

---

# 27. Bootstrap the First Node

The first node needs to create the initial **Primary Component**.

On Node 1:

```bash
sudo systemctl stop mariadb
```

Then bootstrap the cluster:

```bash
sudo galera_new_cluster
```

This starts MariaDB and initializes the first Primary Component.

Check the service:

```bash
sudo systemctl status mariadb
```

Then connect:

```bash
sudo mariadb
```

Check the cluster size:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
```

At this point, the expected result is:

```text
wsrep_cluster_size    1
```

Check cluster status:

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
```

Expected:

```text
Primary
```

Check node state:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

Expected:

```text
Synced
```

<!-- IMAGE PLACEHOLDER: Screenshot of Node 1 after galera_new_cluster -->

---

# 28. Join Node 2

After Node 1 is successfully running as the initial cluster member, configure Node 2 with its own:

```ini
wsrep_node_name
wsrep_node_address
```

Make sure Node 2 can reach Node 1 over the required Galera ports.

Then start MariaDB normally:

```bash
sudo systemctl start mariadb
```

Node 2 should discover the cluster through:

```ini
wsrep_cluster_address="gcomm://10.0.0.11,10.0.0.12,10.0.0.13"
```

Check Node 2:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

It may initially show a joining/synchronizing state.

Wait until it becomes:

```text
Synced
```

Then check:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
```

Expected:

```text
wsrep_cluster_size    2
```

<!-- IMAGE PLACEHOLDER: Screenshot showing Node 2 joined -->

---

# 29. Join Node 3

Configure Node 3:

```ini
wsrep_node_name="mariadb-node3"
wsrep_node_address="10.0.0.13"
```

Then start MariaDB:

```bash
sudo systemctl start mariadb
```

Check:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

Wait until:

```text
Synced
```

Then verify cluster size:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
```

Expected:

```text
wsrep_cluster_size    3
```

The cluster should now look like:

```text
              MariaDB Galera Cluster

        +-----------+-----------+-----------+
        |           |           |           |
        v           v           v
    +-------+   +-------+   +-------+
    |Node 1 |   |Node 2 |   |Node 3 |
    |Synced |   |Synced |   |Synced |
    +-------+   +-------+   +-------+
        \           |           /
         \          |          /
          +---------+---------+
             Galera Network
```

<!-- IMAGE PLACEHOLDER: Screenshot showing 3-node cluster -->

---

# 30. Verify the Complete Cluster

After all nodes have joined, run the following commands on a cluster node.

### Cluster Size

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
```

Expected:

```text
3
```

### Cluster Status

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
```

Expected:

```text
Primary
```

### Node State

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

Expected:

```text
Synced
```

### Cluster Connection

```sql
SHOW STATUS LIKE 'wsrep_connected';
```

Expected:

```text
ON
```

### Node Readiness

```sql
SHOW STATUS LIKE 'wsrep_ready';
```

Expected:

```text
ON
```

A healthy node should generally show:

```text
Cluster Size      = 3
Cluster Status    = Primary
Local State       = Synced
Connected         = ON
Ready             = ON
```

---

# 31. Basic Cluster Health Checklist

After deployment, verify:

```text
[ ] All nodes are running
[ ] All nodes can communicate with each other
[ ] Cluster size is correct
[ ] Cluster status is Primary
[ ] Every node is Synced
[ ] wsrep_connected is ON
[ ] wsrep_ready is ON
[ ] Firewall rules are correct
[ ] MariaDB versions are compatible
[ ] Galera provider is installed
[ ] InnoDB is being used
[ ] binlog_format is ROW
[ ] Replication has been tested
```

<!-- IMAGE PLACEHOLDER: Cluster Health Verification Screenshot -->

---

# 32. Testing Replication

After the cluster is healthy, replication should be tested before considering the deployment complete.

Create a test database on Node 1:

```sql
CREATE DATABASE galera_test;
```

Select the database:

```sql
USE galera_test;
```

Create a table:

```sql
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message VARCHAR(255)
);
```

Insert test data:

```sql
INSERT INTO messages (message)
VALUES ('Hello from Node 1');
```

Now connect to Node 2 and run:

```sql
USE galera_test;

SELECT * FROM messages;
```

Expected:

```text
+----+------------------+
| id | message          |
+----+------------------+
|  1 | Hello from Node 1|
+----+------------------+
```

Now insert another record from Node 2:

```sql
INSERT INTO messages (message)
VALUES ('Hello from Node 2');
```

Finally, check Node 3:

```sql
SELECT * FROM galera_test.messages;
```

Both records should be available:

```text
+----+------------------+
| id | message          |
+----+------------------+
|  1 | Hello from Node 1|
|  2 | Hello from Node 2|
+----+------------------+
```

This confirms that data written on different cluster members is being replicated across the cluster.

<!-- IMAGE PLACEHOLDER: Replication Test — Node 1 -->

<!-- IMAGE PLACEHOLDER: Replication Test — Node 2 -->

<!-- IMAGE PLACEHOLDER: Replication Test — Node 3 -->


## 33. Monitoring and Health Metrics

Monitoring is essential for maintaining the availability, performance, and reliability of a MariaDB Galera Cluster.

A healthy cluster should continuously be monitored for:

* Cluster size
* Cluster state
* Node synchronization state
* Replication connectivity
* Flow control
* Receive and send queues
* Node availability
* Database performance
* Disk and system resources

### 33.1 Cluster Health Checks

Connect to MariaDB and run:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_cluster_status';
SHOW STATUS LIKE 'wsrep_local_state_comment';
SHOW STATUS LIKE 'wsrep_connected';
SHOW STATUS LIKE 'wsrep_ready';
```

A healthy three-node cluster should normally show:

| Metric                      | Expected Value |
| --------------------------- | -------------- |
| `wsrep_cluster_size`        | 3              |
| `wsrep_cluster_status`      | Primary        |
| `wsrep_local_state_comment` | Synced         |
| `wsrep_connected`           | ON             |
| `wsrep_ready`               | ON             |

<!-- IMAGE PLACEHOLDER: MariaDB Galera Cluster Health Status -->

### 33.2 Important Galera Metrics

#### `wsrep_cluster_size`

Shows the number of nodes currently participating in the cluster component.

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
```

For a three-node cluster, the expected value is:

```text
3
```

#### `wsrep_cluster_status`

Shows whether the node belongs to the Primary component.

```sql
SHOW STATUS LIKE 'wsrep_cluster_status';
```

Expected:

```text
Primary
```

#### `wsrep_local_state_comment`

Shows the current synchronization state of the node.

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

Expected:

```text
Synced
```

#### `wsrep_ready`

Indicates whether the node is ready to process database operations.

```sql
SHOW STATUS LIKE 'wsrep_ready';
```

Expected:

```text
ON
```

### 33.3 Flow Control Monitoring

Flow control protects the cluster when one or more nodes cannot process replicated transactions quickly enough.

Useful metrics include:

```sql
SHOW STATUS LIKE 'wsrep_flow_control%';
SHOW STATUS LIKE 'wsrep_local_recv_queue%';
SHOW STATUS LIKE 'wsrep_local_send_queue%';
```

A continuously growing receive queue can indicate that a node is falling behind the rest of the cluster.

Possible causes include:

* Slow disk I/O
* Insufficient CPU
* Memory pressure
* Network latency
* Heavy queries
* Large transactions
* An overloaded node

<!-- IMAGE PLACEHOLDER: Galera Flow Control Monitoring -->

---

## 34. Backup and Disaster Recovery

Galera replication provides high availability, but **replication is not a replacement for backups**.

If incorrect data is inserted, updated, or deleted, the change can be replicated to other nodes. Therefore, independent backups are required for recovery from:

* Accidental deletion
* Application errors
* Data corruption
* Security incidents
* Hardware failure
* Complete cluster failure
* Operational mistakes

### 34.1 Backup Strategy

A production backup strategy should include:

1. Regular full backups
2. Incremental or differential backups where appropriate
3. Off-site backup storage
4. Backup encryption
5. Backup retention policies
6. Regular restore testing
7. Monitoring of backup jobs

### 34.2 Backup Verification

A backup should not be considered reliable simply because the backup command completed successfully.

Regularly test:

```text
Backup
  ↓
Restore
  ↓
Verify Database
  ↓
Verify Application Data
```

<!-- IMAGE PLACEHOLDER: MariaDB Backup and Restore Workflow -->

### 34.3 Disaster Recovery Planning

A disaster recovery plan should define:

* Recovery Point Objective (RPO)
* Recovery Time Objective (RTO)
* Backup location
* Restoration procedure
* Database recovery procedure
* Application recovery procedure
* DNS/load-balancer recovery
* Responsible personnel

A separate disaster recovery environment can be maintained for critical production systems.

---

## 35. Advantages and Limitations

### 35.1 Advantages

MariaDB Galera Cluster provides several important benefits:

* Multi-primary database architecture
* High availability
* Synchronous or virtually synchronous replication
* Automatic node membership management
* Automatic state synchronization
* Reduced dependence on a single database server
* Ability to continue serving traffic after a node failure when quorum is maintained
* Read scalability across multiple nodes
* Automatic recovery mechanisms such as IST and SST

### 35.2 Limitations

Galera Cluster also introduces operational considerations:

* Requires careful network configuration
* Network latency can affect transaction performance
* Write conflicts may occur because multiple nodes can accept writes
* Large transactions can increase replication overhead
* Slow nodes can cause flow control
* Cluster quorum must be carefully designed
* Configuration and troubleshooting are more complex than a standalone MariaDB server
* Galera replication does not replace independent backups

### 35.3 Important Consideration

Adding more database nodes does not automatically provide unlimited write scalability.

Every write must participate in Galera's replication and certification process. Therefore, application workload, transaction size, network latency, and node performance must be considered when designing the cluster.

---

## 36. Troubleshooting

Troubleshooting should begin by determining whether the problem is related to:

* Node availability
* Network connectivity
* Cluster membership
* Synchronization
* Flow control
* Database performance
* Storage
* Configuration

### 36.1 Check Cluster Status

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_cluster_status';
SHOW STATUS LIKE 'wsrep_local_state_comment';
SHOW STATUS LIKE 'wsrep_connected';
SHOW STATUS LIKE 'wsrep_ready';
```

### 36.2 Check MariaDB Service

```bash
sudo systemctl status mariadb
```

Restart when required:

```bash
sudo systemctl restart mariadb
```

### 36.3 Check Galera Logs

Depending on the MariaDB configuration, inspect the MariaDB error log:

```bash
sudo journalctl -u mariadb
```

For recent messages:

```bash
sudo journalctl -u mariadb -n 100
```

### 36.4 Node Is Not Joining the Cluster

Check:

1. MariaDB service status
2. Galera configuration
3. Node IP address
4. Cluster address
5. Firewall rules
6. Galera ports
7. DNS/hostname resolution
8. MariaDB and Galera versions
9. Error logs

Verify network connectivity between nodes.

For example:

```bash
ping 10.0.0.12
```

Check listening ports:

```bash
sudo ss -lntup
```

### 36.5 Cluster Is in Non-Primary State

If a node reports:

```text
Non-Primary
```

the node is not currently part of a Primary Component.

Do not immediately bootstrap a new cluster.

First determine:

* Whether other nodes are still running
* Which node contains the current Primary Component
* Whether the network partition has been resolved
* Whether quorum has been maintained

Bootstrapping a new Primary Component should be treated as a recovery operation because starting an independent component incorrectly can result in a split-brain situation.

### 36.6 Node Is Stuck in Donor or Desynced State

Check:

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
SHOW STATUS LIKE 'wsrep_local_recv_queue%';
SHOW STATUS LIKE 'wsrep_local_send_queue%';
```

Investigate:

* Disk performance
* CPU usage
* Memory pressure
* Network performance
* Large transactions
* Database workload

### 36.7 IST or SST Problems

If a joining node cannot receive the required incremental state, Galera may require a State Snapshot Transfer (SST).

Check:

* Donor node availability
* SST configuration
* Port `4444`
* IST port `4568`
* Authentication configuration
* Available disk space
* MariaDB logs

Example:

```bash
sudo journalctl -u mariadb -n 200
```

<!-- IMAGE PLACEHOLDER: Galera Troubleshooting / Error Log Screenshot -->

### 36.8 Common Troubleshooting Checklist

```text
[ ] MariaDB service is running
[ ] Node has correct IP address
[ ] Cluster address is correct
[ ] Galera provider is installed
[ ] Galera ports are reachable
[ ] Firewall allows required traffic
[ ] wsrep_on is enabled
[ ] Node is connected
[ ] Cluster is Primary
[ ] Node state is Synced
[ ] Disk space is available
[ ] System resources are sufficient
[ ] MariaDB logs show no critical errors
```

---

## 37. Recommended Production Architecture

A production MariaDB Galera deployment should separate database availability from application traffic management.

A typical architecture is:

```text
                         Users / Applications
                                  |
                                  v
                         +------------------+
                         | Load Balancer    |
                         +------------------+
                                  |
                    +-------------+-------------+
                    |             |             |
                    v             v             v
              +---------+   +---------+   +---------+
              | Node 1  |   | Node 2  |   | Node 3  |
              | MariaDB |<->| MariaDB |<->| MariaDB |
              | Galera  |   | Galera  |   | Galera  |
              +---------+   +---------+   +---------+
                    |             |             |
                    +-------------+-------------+
                                  |
                           Backup System
```

<!-- IMAGE PLACEHOLDER: Recommended Production MariaDB Galera Architecture -->

### 37.1 Application Layer

Applications should connect through a database access layer or load-balancing mechanism rather than depending directly on a single database node.

This allows database traffic to be distributed and provides better failure handling.

### 37.2 Database Layer

The database layer should contain multiple Galera nodes distributed across suitable failure domains.

For example:

```text
Database Node 1 → Availability Zone A
Database Node 2 → Availability Zone B
Database Node 3 → Availability Zone C
```

The exact design depends on the infrastructure and network topology.

### 37.3 Backup Layer

Backups should be stored independently from the database nodes.

For example:

```text
MariaDB Cluster
      |
      v
Backup System
      |
      +---- Local Backup
      |
      +---- Remote Backup
      |
      +---- Off-site / Object Storage
```

### 37.4 Monitoring Layer

Production deployments should also include monitoring and alerting for:

* Node availability
* Cluster size
* Primary component status
* Synchronization state
* Flow control
* Replication queues
* CPU
* Memory
* Disk usage
* Disk latency
* Network performance
* Backup success/failure

---

## 38. Final Conclusion

MariaDB Galera Cluster provides a high-availability database architecture based on multi-primary replication and automatic cluster membership management.

Its major components include:

* Multi-primary architecture
* Galera replication
* Certification-based transaction processing
* Quorum
* State synchronization
* IST
* SST
* Flow control
* Automatic node recovery mechanisms

A reliable production deployment requires more than simply configuring three MariaDB servers. Network design, quorum, security, monitoring, backups, performance, and disaster recovery must all be considered.

The basic operational model can be summarized as:

```text
Plan Infrastructure
        ↓
Install MariaDB + Galera
        ↓
Configure Cluster
        ↓
Bootstrap First Node
        ↓
Join Additional Nodes
        ↓
Verify Cluster Health
        ↓
Test Replication
        ↓
Configure Monitoring
        ↓
Configure Backups
        ↓
Deploy Behind Load Balancer
        ↓
Operate and Monitor
```

MariaDB Galera Cluster is therefore best understood as a complete high-availability database architecture rather than simply a replication feature.

---

## 39. References

The documentation and concepts in this guide should be validated against the official MariaDB documentation for the specific MariaDB and Galera versions being deployed.

Recommended reference areas:

* MariaDB Galera Cluster documentation
* Galera Cluster concepts
* Galera configuration parameters
* Galera status variables
* State Snapshot Transfer (SST)
* Incremental State Transfer (IST)
* Flow Control
* MariaDB backup and recovery documentation
* MariaDB security and TLS documentation
* MariaDB MaxScale documentation

### Recommended Documentation Sources

**MariaDB Documentation**
Official documentation for MariaDB Server, Galera Cluster, configuration, administration, backup, and recovery.

**MariaDB Galera Cluster Documentation**
Detailed information about Galera architecture, cluster configuration, node states, replication, and recovery.

**MariaDB MaxScale Documentation**
Documentation for database proxying, routing, monitoring, and high-availability architectures.

<!-- IMAGE PLACEHOLDER: Final MariaDB Galera Cluster Documentation / Architecture Overview -->

