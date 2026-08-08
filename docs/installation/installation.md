# Download &amp; Install

This page walks through downloading the ShannonStore distribution and bringing up an evaluation cluster on a single machine. For a guided first-object workflow with the AWS CLI, continue to [Getting Started](getting-started.md). For ongoing operations of an existing cluster, see [Cluster Operations](../operations/operations.md).

## Prerequisites

- **Java 17** — ShannonStore is written in Java and ships as a tarball; Java 17 (or newer) must be on `PATH`.
- **A strong master key (32+ characters recommended)** — used to derive the cluster's root encryption key. Set as the environment variable `SHANNONSTORE_MASTER_KEY` before starting any node. The only thing the process actually enforces at startup is that the variable is set and non-blank (`ClusterKmsProvider` fails closed with a `SecurityException` otherwise) — there is no minimum-length check in code, so the 32-character guidance is an operational recommendation, not an enforced requirement.
- **macOS or Linux** — the supplied start scripts are bash and assume a POSIX shell.

## Download the Release Archive

```agsl
curl -L -O https://github.com/cloudcheflabs/shannonstore-pack/releases/download/shannonstore-archive/shannonstore-1.0.0.tar.gz
```

Extract the archive and change into the resulting directory:

```agsl
tar zxvf shannonstore-1.0.0.tar.gz
cd shannonstore-1.0.0
```

The archive ships with the layout below:

| Path | Contents |
|---|---|
| `bin/` | Start/stop scripts for ZooKeeper, API Servers, and Data Nodes |
| `lib/` | All JARs needed to run the API Server and Data Node main classes |
| `conf/` | `shannonstore.properties` defaults and `logback.xml` |
| `admin-ui/` | Built admin SPA, served by the API Server's admin port |
| `data/` | Default data directory used by Data Nodes (`data/node1-disk-a`, etc.) |
| `logs/` | Default log destination |

## Start an Evaluation Cluster

The bundled helper script brings up an embedded ZooKeeper, three Data Nodes, and one API Server in a single command — useful for kicking the tires on a laptop:

```agsl
export SHANNONSTORE_MASTER_KEY="ShannonStoreMasterKey1200303003abc"
bin/start-example-servers.sh
```

> A 32+ character master key is recommended (not enforced by the process — only presence is checked). The same value must be supplied to **every** ShannonStore process (API or Data) in a real deployment — losing it means losing access to all encrypted data.

After a few seconds the cluster is ready. The API Server listens on:

| Port | Purpose |
|---|---|
| `8080` | S3 API — point S3 clients here |
| `8888` | Admin UI and admin API (browse `/admin`) |
| `9000` | Internal cluster communication |

Data Nodes use ports `9001`, `9002`, `9003`, each with two storage directories on local disk.

To stop the example cluster:

```agsl
bin/stop-example-servers.sh
```

## Running Individual Roles

For a real deployment you start each role separately and point them at an external ZooKeeper. Each script reads `conf/shannonstore.properties` and accepts `-D` overrides on the command line.

### ZooKeeper

```agsl
bin/start-zk.sh
```

Production deployments usually point at a separately-managed ensemble. Set the connect string via `shannonstore.zk.connect` in the properties file or the environment variable `SHANNONSTORE_ZK_CONNECT`.

### API Server

```agsl
bin/start-api-server.sh \
  -Dshannonstore.api.s3.port=8080 \
  -Dshannonstore.api.admin.port=8888 \
  -Dshannonstore.nio.port=9000
```

Repeat with different port triples on each host (or container) to scale the API tier horizontally. The cluster automatically elects exactly one API Server as leader.

### Data Node

```agsl
bin/start-data-node.sh \
  -Dshannonstore.nio.port=9001 \
  -Dshannonstore.data.storage.dirs=/var/lib/shannonstore/disk-a,/var/lib/shannonstore/disk-b
```

Each Data Node can pack chunks across multiple storage directories — typically one per disk — to balance load and tolerate single-disk failures. See [Multi-Disk Storage](../features/multi-disk-storage.md) for details.

### Stopping individual roles

Each `start-*.sh` writes a PID file under `bin/`, and a matching `stop-*.sh` reads it to shut the process down gracefully. Use these in your service manager (systemd, k8s lifecycle hooks, etc.) instead of sending raw signals.

## Verifying the Cluster

Once nodes are up, the admin port exposes a health endpoint:

```agsl
curl http://localhost:8888/admin/health
```

While the cluster is still starting up, this endpoint reports the node as not ready. Once every API Server has synchronised state from the leader and at least one Data Node is registered, the endpoint reports the node as ready and the S3 port begins accepting client requests.

For the next step — creating a bucket, an access key, and uploading an object — continue to [Getting Started](getting-started.md).
