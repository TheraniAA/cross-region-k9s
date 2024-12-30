# Distributed ClickHouse with ZooKeeper Failover Setup

This repository contains configuration and instructions for setting up a distributed ClickHouse cluster with ZooKeeper for high availability and failover capabilities. The setup includes two clusters with primary-secondary configuration.

## Architecture Overview

- **Cluster-1 (Primary)**: Contains primary ZooKeeper ensemble and ClickHouse nodes
- **Cluster-2 (Secondary)**: Contains observer ZooKeeper ensemble and ClickHouse nodes
- Automated failover capability between clusters
- Support for tailback operations

## Prerequisites

- Kubernetes cluster with kubectl configured
- k9s installed (optional, for cluster monitoring)
- ClickHouse client
- Access to both clusters with proper context configuration

## Cluster Setup

### Cluster-1 (Primary) Setup

```bash
# Switch to Cluster-1 context
kubectl config use-context cluster-1

# Deploy ZooKeeper and ClickHouse nodes
kubectl apply -f zk-3-primary-pvc.yml
kubectl apply -f ord-3-nodes.yaml

# Verify ClickHouse connectivity
kubectl port-forward service/clickhouse-ord 9001:tcp &
clickhouse client --port 9001
```

### Cluster-2 (Secondary) Setup

```bash
# Switch to Cluster-2 context
kubectl config use-context cluster-2

# Deploy ZooKeeper and ClickHouse nodes
kubectl apply -f zk-3-observer-secondary-pvc.yml
kubectl apply -f sfo-3-nodes.yaml

# Verify ClickHouse connectivity
kubectl port-forward service/clickhouse-sfo 9002:tcp &
clickhouse client --port 9002
```

## Verification and Monitoring

### Check ZooKeeper Status

```bash
# Connect to ZooKeeper
echo srvr | nc localhost 2181

# Check ZooKeeper configuration
zkCli.sh -server localhost:2181
get /zookeeper/config
```

### Create Test Tables and Data

```sql
-- Create distributed table
CREATE TABLE default.shared5v1 ON CLUSTER umbrella_cluster
(
    `id` UInt64,
    `created_at` DateTime DEFAULT now(),
    `value` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/shared5v1', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (id, created_at);

-- Insert test data
INSERT INTO default.shared5v1 (id, value) FORMAT Values (1, 'umbrella_cluster');
```

## Failover Procedure

1. **Initiate Failover on Cluster-1**
```bash
kubectl config use-context cluster-1
kubectl delete -f ord-3-nodes.yaml
kubectl delete -f zk-3-primary-pvc.yml
```

2. **Verify Cluster-2 Status**
```bash
kubectl config use-context cluster-2
echo srvr | nc localhost 2181
```

3. **Apply Recovery Configuration**
```bash
kubectl apply -f zk-3-observer-secondary-pvc-recover.yml
```

## Tailback Operation

The tailback operation allows you to switch back to the original configuration after resolving issues in the primary cluster.

```bash
# On Cluster-2
kubectl config use-context cluster-2
kubectl apply -f zk-3-observer-secondary-tailback-pvc.yml

# On Cluster-1
kubectl config use-context cluster-1
kubectl apply -f zk-3-primary-tailback-pvc.yml
kubectl apply -f ord-3-nodes.yaml
```

## Important Notes

- After tailback, your secondary site becomes primary and vice versa
- Always verify ZooKeeper ensemble status before and after operations
- Monitor ClickHouse replication status using system tables
- Maintain proper backup procedures before performing failover operations

## Configuration Files

- `zk-3-primary-pvc.yml`: Primary ZooKeeper configuration
- `ord-3-nodes.yaml`: ClickHouse nodes configuration for Cluster-1
- `zk-3-observer-secondary-pvc.yml`: Secondary ZooKeeper configuration
- `sfo-3-nodes.yaml`: ClickHouse nodes configuration for Cluster-2
- `zk-3-observer-secondary-pvc-recover.yml`: Recovery configuration
- `zk-3-observer-secondary-tailback-pvc.yml`: Secondary cluster tailback configuration
- `zk-3-primary-tailback-pvc.yml`: Primary cluster tailback configuration

## Troubleshooting

1. Check ZooKeeper connection status:
```sql
SELECT * FROM system.zookeeper_connection;
```

2. Verify cluster health:
```bash
kubectl config use-context <cluster-name>
k9s
```

3. Monitor ZooKeeper ensemble state:
```bash
echo mntr | nc localhost 2181 | grep zk_server_state
```
