# CockroachDB — Two-Node Deployment

**Project:** Pace University CS371
**Contributors:** Bayron Arevalo Loja (Ryzen-0) and Justin O'Brien (Twix)
**Last verified:** November 18–19, 2026
**Status:** Functional SQL cluster; data replication incomplete

## Goal

Deploy CockroachDB across two Ryzen servers connected through the Nebula VPN.

The objective is to establish a distributed SQL environment, verify connectivity and SQL functionality from both machines, and evaluate data replication.

## Prerequisites

- Two Linux servers with Docker installed
- Nebula VPN connectivity
- CockroachDB v25.4.4
- TLS certificates for secure communication
- Persistent Docker storage volumes
- SSH access to both machines

### Server Configuration

| Server | Hostname | Nebula IP | Node ID |
|---|---|---|---|
| Ryzen-0 | cs371-bc9b0c | 10.80.0.10 | 1 |
| Twix | cs371-bc9a92 | 10.80.0.11 | 2 |

## Design Decisions

We deployed CockroachDB using standalone Docker containers with host networking.

Each server has its own persistent Docker volume named `cockroach-data`, mounted inside the container at `/cockroach/cockroach-data`.

TLS certificates are mounted at `/cockroach/certs`.

The deployment uses CockroachDB's default three-replica target. Our current two-node environment does not satisfy this requirement.

## Installation and Configuration

CockroachDB runs in a Docker container named `cockroachdb` on each server.

### Certificate Locations

**Ryzen-0:**

`/home/milkalmond/cockroach-cluster/ryzen0-certs`

**Twix:**

`/home/twix/cockroach-cluster/certs`

Both directories are mounted inside their respective containers at `/cockroach/certs`.

Required certificates include:

- `ca.crt`
- `node.crt`
- `node.key`
- `client.root.crt`
- `client.root.key`

The root client certificate and key are required for certificate-based administrative SQL access. Private keys must be protected and never committed to GitHub.

## Verification

### 1. Verify Cluster Membership

On Ryzen-0:

```bash
docker exec cockroachdb \
  ./cockroach node status \
  --certs-dir=/cockroach/certs \
  --host=10.80.0.10:26257
```

On Twix:

```bash
docker exec cockroachdb \
  ./cockroach node status \

 --certs-dir=/cockroach/certs \
  --host=10.80.0.11:26257

Result: Both nodes reported is_available=true and is_live=true.

## 2. Verify SQL Connectivity

On Twix:

docker exec cockroachdb \
  ./cockroach sql \
  --certs-dir=/cockroach/certs \
  --host=10.80.0.11:26257 \
  -e "SHOW DATABASES;"

Result: The following databases were visible:

Database           Owner

cs371_test         root

defaultdb          root

postgres           root

system             node

This confirms that Twix can execute SQL queries against the CockroachDB cluster.

## 3. Verify Replica Distribution

Run:

docker exec cockroachdb \
  ./cockroach sql \
  --certs-dir=/cockroach/certs \
  --host=10.80.0.11:26257 \
  -e "SELECT node_id, store_id,
      metrics->>'replicas' AS replicas,
      metrics->>'ranges.underreplicated' AS underreplicated
      FROM crdb_internal.kv_store_status;"

Observed results:

Node      Store     Replicas    Under-replicated

Ryzen-0 (1)  1         55              55

Twix (2)     2         0                0

All 55 ranges were located on Ryzen-0, with no replicas on Twix.

Important: Although both servers are operational and SQL connectivity works, this deployment does not currently provide data redundancy or high availability.

Troubleshooting

Missing Root Client Certificate on Twix

Initially, CockroachDB administrative commands on Twix prompted for a root password and failed with:

SQLSTATE: 28P01

Inspection showed that Twix was missing client.root.crt and client.root.key.

We securely transferred the root client certificate and key from Ryzen-0 to Twix and restricted private-key permissions to 600.

Afterward, the cockroach node status and SHOW DATABASES commands worked without password prompts.

Data Replication Not Working

CockroachDB logs reported:

avoid up-replicating to fragile quorum
likely not enough nodes in cluster

The cluster currently has two nodes, while the default configuration targets three replicas.

Replica checks confirmed that all 55 ranges remained on Ryzen-0.

We have not changed the replication factor or removed any existing data.

Next Steps

Coordinate with Alex to obtain approval for a third independent server.

Review the cockroachdb-sandbox reference deployment for Docker Swarm and HAProxy.

Plan a three-node CockroachDB deployment with persistent storage and TLS.

Verify that replicas are distributed across nodes.

Perform controlled failover testing after replication is healthy.

Conclusion

We successfully deployed and verified a two-node CockroachDB cluster across Ryzen-0 and Twix.

Both machines can access the same SQL cluster using TLS authentication.

However, the current deployment is suitable for development and SQL connectivity testing only. Data replication and high availability remain incomplete.
