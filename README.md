# kafka (local role)

Installs **Apache Kafka 4.x** (vanilla) in **KRaft combined mode** —
one process per node running `process.roles=broker,controller`. No ZooKeeper, no
SASL/TLS (PLAINTEXT), no external repos.

Steps: create `kafka` user + dirs → download/unpack the Apache tarball → render
`server.properties` (KRaft, PLAINTEXT, RF=3/min.insync=2, `unclean.leader.election=false`,
`auto.create.topics.enable=false`) → `kafka-storage.sh format` once (guarded by
`meta.properties`) → systemd unit → start.

`node.id` and `controller.quorum.voters` come from the `kafka` inventory group
(`kafka_node_id` + `ansible_host` per host). The cluster id is `kafka_cluster_id`
(group_vars) — identical on all nodes.

Requires Java 17+ (installed by the playbook's `pre-tasks/java.yaml`) and the data
disk mounted at `/var/lib/kafka` (by the `manage-lvm` role) before this role runs.

**Topics are not created** (names not finalized). Create manually once agreed:

```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic <name> --partitions 3 --replication-factor 3 \
  --config min.insync.replicas=2
```

**Rolling restart:** a config change notifies a restart on all nodes. For a live
cluster restart one at a time (`--limit stage-did-kafka-0N`) and wait for
`--under-replicated-partitions` to clear between nodes.
