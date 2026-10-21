# Project Specification: Highly Available PostgreSQL Cluster

## 1 Overview

A single database is a single point of failure: if the server goes down, the
application stops. This project therefore builds a PostgreSQL cluster that
survives the failure of a node automatically, without manual intervention and
without losing any committed transactions.

The cluster consists of three PostgreSQL nodes (one leader, two replicas)
managed by Patroni, one etcd node for leader election, and an HAProxy as the
single entry point for the application. The entire setup is automated with
Ansible.

## 2 Goals

|    | Goal |
|----|------|
| **1** | Three nodes: exactly one writable leader, two read-only replicas. |
| **2** | Automatic failover within ≤ 60 seconds when the leader fails. |
| **3** | No loss of committed transactions (RPO = 0). |
| **4** | The application connects to a single, stable endpoint. |
| **5** | Read traffic goes to the replicas, write traffic to the leader. |
| **6** | A returning node automatically rejoins as a replica. |
| **7** | The setup is reproducible and idempotent with Ansible. |
| **8** | No split-brain: two nodes never accept writes at the same time. |


## 3 Architecture

![Architecture diagram of the HA PostgreSQL cluster](assets/schema.jpg)

```
Application
     │
     ▼
  HAProxy  ──── :5000 (read/write) --> Leader
           └─── :5001 (read only)  --> Replica 1, Replica 2

  pg01 <-- streaming replication ── pg02, pg03
  pg01, pg02, pg03  <--- Leader-Key --->  etcd01
```

### 3.1 Components

**HAProxy** is the single entry point. It asks each node whether it is
currently the leader or a replica, and routes writes (port 5000) to the leader
and reads (port 5001) to the replicas. After a failover it follows the new
role assignment on its own.

**PostgreSQL nodes.** There is always exactly one leader, which accepts all
writes, and two replicas, which continuously receive its data. If the leader
fails, a replica becomes the new leader. The old leader later comes back as a
replica.

**etcd** is the shared source of truth: it holds a leader key that the current
leader must renew regularly. If the leader stops renewing it, the key expires
and a replica takes over. This ensures there are never two leaders. This
project uses only one etcd node. Ideally there would be several, otherwise the
etcd node becomes a single point of failure.

**Patroni** runs on every PostgreSQL node and acts as the cluster manager: it
starts and stops PostgreSQL, holds the leader key in etcd, decides which
replica takes over on failure, and reintegrates returning nodes.

### 3.2 Nodes

| Hostname | Role | Ansible group |
|----------|------|---------------|
| `haproxy01` | Reverse proxy / load balancer | `haproxy` |
| `pg01` | PostgreSQL + Patroni | `patroni` |
| `pg02` | PostgreSQL + Patroni | `patroni` |
| `pg03` | PostgreSQL + Patroni | `patroni` |
| `etcd01` | etcd | `etcd` |

All nodes run Debian 13 and are reachable via SSH. The Ansible control node is
the workstation and is not part of the cluster.

### 3.3 What happens on failure

When the leader fails, it stops renewing the leader key in etcd. After at most
30 seconds (TTL) the key expires, the remaining nodes notice, and **Patroni
promotes the most up-to-date replica to the new leader**. HAProxy detects the
change through its health checks and routes writes to the new leader from
that moment on. Overall this takes about one minute.

When the failed node starts again later, Patroni sees that another leader
already exists and automatically attaches it as a replica.

### 3.4 Key decisions

| Decision | Choice | Rationale |
|---|---|---|
| Replication | synchronous | The only way to guarantee no committed transactions are lost (RPO = 0). |
| DCS | etcd | Lightweight and best supported by Patroni. |
| Connection routing | HAProxy | The cluster itself reports who the leader is. No second source of truth. |
| Failback | none | A returning node stays a replica; this avoids a second interruption. |
| etcd nodes | only one | Deliberate simplification for this project, see section 5. |


## 4 Tests

The cluster is mainly verified through Patroni: **Patroni is used to check
whether the databases are running** and which node is currently leader or
replica. In addition, connections are made through HAProxy and a node is
deliberately shut down to observe the failover.

|    | Test | How it is checked | Expected result |
|----|------|-------------------|-----------------|
| **1** | Deployment | Run the playbook twice in a row and read the Ansible recap of the second run. | The playbook completes without errors; a second run changes nothing. |
| **2** | Check cluster state with Patroni | Run `patronictl list` in a terminal on a PG node and read the role and state columns of the output. | One leader, two replicas, all running. |
| **3** | Replication | Write to a test table with `psql` via HAProxy port 5000, then query the same row directly on both replicas. | New data is visible on both replicas shortly afterwards. |
| **4** | Failover | Hard power off the leader VM while watching `patronictl list` and logging timestamped write attempts against port 5000 in a loop. | Leader powered off: a replica takes over automatically, writes work again after about one minute, nothing is lost. |
| **5** | Rejoin | Start the failed node again and then check `patronictl list`. | The restarted node comes back as a replica on its own. |
| **6** | Planned role change | Run `patronictl switchover` while keeping a write connection open via port 5000. | Switchover with Patroni succeeds with only a brief interruption. |
| **7** | Endpoints | Run one `psql` query each via port 5000 and port 5001 that returns the hostname and read-only status of the node reached. | Port 5000 always reaches the leader, port 5001 always a replica. |
| **8** | etcd failure | Stop the etcd service, then test reads and writes via HAProxy, start etcd again and test again. | The cluster becomes read-only and recovers as soon as etcd is back. |
| **9** | Split-brain | Isolate the leader from the rest of the network with a firewall rule and attempt writes on both sides of the partition. | Even during a network partition, no more than one node ever accepts writes. |
| **10** | Scaling out | Add a fourth node to the Ansible inventory, run the playbook and check `patronictl list`. | An additional node added via the inventory appears as another replica. |

Each test will be documented in the project report with command, output and
timestamp. The failover tests will be run via scripts under `tests/` so that the
measurement is reproducible.


## 5 Known limitations

|   | Limitation | Handling |
|---|------------|----------|
| 1 | etcd runs on only one node and is therefore a single point of failure. | Deliberately accepted. Production would require three etcd nodes. |
| 2 | HAProxy is also a single point of failure. | Out of project scope; solution in the outlook. |
| 3 | Synchronous replication costs write performance. | Deliberate price for RPO = 0. |
| 4 | Short network interruptions can trigger unnecessary role changes. | Patroni defaults instead of tuning for the shortest possible downtime. |


## 6 Outlook

- etcd as a three-node cluster, the most important step towards production.
- Redundant HAProxy with Keepalived and a virtual IP.
- Backups with Barman or pgBackRest: high availability does not protect
  against an accidental `DROP TABLE`.
- Monitoring of replication, leader changes and etcd.
- Connection pooling with PgBouncer.


## 7 Glossary

| Term | Meaning |
|------|---------|
| **Leader / Primary** | The node that accepts writes. Exactly one per cluster. |
| **Replica / Standby** | A node that receives the leader's data and only allows reads. |
| **Failover** | Unplanned role change after a failure. |
| **Switchover** | Planned, controlled role change. |
| **Recovery Time Objective (RTO)** | Maximum tolerated downtime. |
| **Recovery Point Objective (RPO)** | Maximum tolerated data loss; 0 means no committed transaction is lost. |
| **Distributed Configuration Store (DCS)** | Distributed store for the cluster state, here etcd. |
| **Split-Brain** | Failure state in which two nodes accept writes at the same time. |
