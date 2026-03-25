# Containerized Kafka Cluster

Crude, but semi-clean containerized Kafka cluster for prototyping and
teaching purposes.  There are a couple of niceties I want to add to
it, but they'll come later.

## Quick Start

To get the cluster running:
```
docker compose up -d
```

## Destroying the Cluster

To get rid of the cluster:
```
docker compose down
```

## Getting a Bash Shell on One of the Cluster Nodes

```
docker compose exec -it [node service name] /bin/bash
```
The container image stores the usual Kafka command line utilities in `/usr/bin`.

## Fiddling with the Zookeeper Shell

```
docker compose exec -it [kafka node service name] /bin/bash

zookeeper-shell $ZOOKEEPER
```
You'll end up with a barebones JLine-free minimally usable ZK shell.

## Nice to Haves to Add Later

- **Data Storage:** Mount host volumes in the containers to use as the cluster's data
directories; also add some scripts for deleting the contents thereof
during a full teardown.
- Possibly add **Kafbat UI** (`kafbat/kafka-ui`) container for web-based cluster management
   - **Note:** Use `kafbat/kafka-ui`, NOT `provectus/kafka-ui` (abandoned with known security vulnerabilities)
   - [Kafbat UI on GitHub](https://github.com/kafbat/kafka-ui) - actively maintained fork
   - [Docker Compose examples](https://ui.docs.kafbat.io/configuration/compose-examples)
   - Provides web UI for browsing topics, messages, consumer groups, and cluster metrics
   - Simple integration: just needs bootstrap servers and optionally Zookeeper connection
- Possibly add **Confluent Schema Registry** (`confluentinc/cp-schema-registry`)
   - Centralized schema management for Avro, Protobuf, and JSON schemas
   - Version control and compatibility checking for message schemas
   - Integrates with Kafbat UI for schema browsing
   - Uses Kafka itself as storage backend (via `_schemas` topic)
- Possibly add some of the Confluent platform images from
  `confluentinc/cp-*` on Docker Hub, assuming I end up needing them
  for what I ultimately use this for.
