# Distributed ClickHouse with ZooKeeper Multi-Cluster Setup

This repository contains detailed instructions and configurations for setting up a distributed ClickHouse deployment with ZooKeeper for high availability, including failover and tailback procedures.

## Required Files

- `zk-3-primary-pvc.yml`
- `ord-3-nodes.yaml`
- `zk-3-observer-secondary-pvc.yml`
- `sfo-3-nodes.yaml`
- `zk-3-observer-secondary-pvc-recover.yml`
- `zk-3-observer-secondary-tailback-pvc.yml`
- `zk-3-primary-tailback-pvc.yml`

## Initial Setup

### Cluster-1 (Primary) Setup

```bash
# Switch context to cluster-1
kubectl config use-context cluster-1

# Deploy ZooKeeper and ClickHouse
kubectl apply -f zk-3-primary-pvc.yml
kubectl apply -f ord-3-nodes.yaml

# Verify ClickHouse Connection
kubectl port-forward service/clickhouse-ord 9001:tcp &
clickhouse client --port 9001
select * from system.zookeeper_connection;
```

### Cluster-2 (Secondary) Setup

```bash
# Switch context to cluster-2
kubectl config use-context cluster-2

# Deploy ZooKeeper and ClickHouse
kubectl apply -f zk-3-observer-secondary-pvc.yml
kubectl apply -f sfo-3-nodes.yaml

# Verify ClickHouse Connection
kubectl port-forward service/clickhouse-sfo 9002:tcp &
clickhouse client --port 9002
select * from system.zookeeper_connection;
```

## Data Management

### Create and Load Sample Tables

```sql
-- Create distributed table
DROP TABLE IF EXISTS default.shared5v1 ON CLUSTER umbrella_cluster sync;
CREATE TABLE default.shared5v1 ON CLUSTER umbrella_cluster
(
    `id` UInt64,
    `created_at` DateTime DEFAULT now(),
    `value` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/shared5v1', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (id, created_at);

-- Insert sample data
INSERT INTO default.shared5v1 (id, value) FORMAT Values (1, 'umbrella_cluster');
INSERT INTO default.shared5v1 (id, value) FORMAT Values (2, 'primary_cluster');
INSERT INTO default.shared5v1 (id, value) FORMAT Values (3, 'secondary_cluster');
```

### Additional Table Example

```sql
DROP TABLE IF EXISTS default.shared5v123 ON CLUSTER umbrella_cluster sync;
CREATE TABLE default.shared5v123 ON CLUSTER umbrella_cluster
(
    `id` UInt64,
    `created_at` DateTime DEFAULT now(),
    `value` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/shared5v123', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (id, created_at);

INSERT INTO default.shared5v123 (id, value) FORMAT Values (1, 'umbrella');
```

## ZooKeeper Configuration

### Check Cluster Status
```bash
# View ZooKeeper status
echo srvr | nc localhost 2181

# Connect to ZooKeeper CLI
zkCli.sh -server localhost:2181
get /zookeeper/config
```

### Initial ZooKeeper Configuration
```
server.1=zookeeper-ord-0-0.zookeeper-ord.default.svc.cluster.local:2888:3181:participant;0.0.0.0:2181
server.2=zookeeper-ord-1-0.zookeeper-ord.default.svc.cluster.local:2888:3181:participant;0.0.0.0:2181
server.3=zookeeper-ord-2-0.zookeeper-ord.default.svc.cluster.local:2888:3181:participant;0.0.0.0:2181
server.4=zookeeper-sfo-0-0.zookeeper-sfo.default.svc.cluster.local:2888:3181:observer;0.0.0.0:2181
server.5=zookeeper-sfo-1-0.zookeeper-sfo.default.svc.cluster.local:2888:3181:observer;0.0.0.0:2181
server.6=zookeeper-sfo-2-0.zookeeper-sfo.default.svc.cluster.local:2888:3181:observer;0.0.0.0:2181
```

## Failover Procedure

1. **Initiate Failover on Cluster-1**
```bash
kubectl config use-context cluster-1
kubectl delete -f ord-3-nodes.yaml
kubectl delete -f zk-3-primary-pvc.yml
```

2. **Verify Cluster Status**
```bash
show tables;
select * from system.zookeeper_connection;
```

3. **Check Cluster-2 Status**
```bash
kubectl config use-context cluster-2
echo srvr | nc localhost 2181
zkCli.sh -server localhost:2181
get /zookeeper/config
```

4. **Apply Recovery Configuration**
```bash
kubectl config use-context cluster-2
kubectl apply -f zk-3-observer-secondary-pvc-recover.yml
clickhouse client --port 9002

# Verify recovery
show tables;
select * from system.zookeeper_connection;
```

## Tailback Procedure

1. **Prepare Secondary Cluster**
```bash
kubectl config use-context cluster-2
kubectl apply -f zk-3-observer-secondary-tailback-pvc.yml
```

2. **Restore Primary Cluster**
```bash
kubectl config use-context cluster-1
kubectl apply -f zk-3-primary-tailback-pvc.yml
kubectl apply -f ord-3-nodes.yaml
```

**Important Note:** After tailback completion, your secondary site becomes primary and vice versa.

### Post-Tailback Configuration
```
server.1=zookeeper-sfo-0-0.zookeeper-sfo.default.svc.cluster.local:2888:3888:participant;0.0.0.0:2181
server.2=zookeeper-sfo-1-0.zookeeper-sfo.default.svc.cluster.local:2888:3888:participant;0.0.0.0:2181
server.3=zookeeper-sfo-2-0.zookeeper-sfo.default.svc.cluster.local:2888:3888:participant;0.0.0.0:2181
server.4=zookeeper-ord-0-0.zookeeper-ord.default.svc.cluster.local:2888:3888:observer;0.0.0.0:2181
server.5=zookeeper-ord-1-0.zookeeper-ord.default.svc.cluster.local:2888:3888:observer;0.0.0.0:2181
server.6=zookeeper-ord-2-0.zookeeper-ord.default.svc.cluster.local:2888:3888:observer;0.0.0.0:2181
```

## Monitoring and Verification

### Check ZooKeeper State
```bash
echo mntr | nc localhost 2181 | grep zk_server_state
```

### View ZooKeeper Configuration
```bash
kubectl exec -it zookeeper-ord-0-0 -- cat /opt/bitnami/zookeeper/conf/zoo.cfg
kubectl exec -it zookeeper-sfo-0-0 -- cat /opt/bitnami/zookeeper/conf/zoo.cfg
```

## Secondary Cluster Operations

### Create Secondary Cluster Table
```sql
DROP TABLE IF EXISTS default.sharedAAA ON CLUSTER secondary_cluster sync;
CREATE TABLE default.sharedAAA ON CLUSTER secondary_cluster
(
    `id` UInt64,
    `created_at` DateTime DEFAULT now(),
    `value` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/sharedAAA', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (id, created_at);

-- Insert test data
INSERT INTO default.sharedAAA (id, value) FORMAT Values (1, 'secondary_cluster');

-- Query all replicas
SELECT * FROM clusterAllReplicas('secondary_cluster', default.sharedAAA);
```

## Troubleshooting

1. Always verify ZooKeeper connection status after configuration changes
2. Monitor cluster health using k9s
3. Check ZooKeeper ensemble state regularly
4. Verify table replication status across clusters
5. Monitor port forwarding status for ClickHouse connections