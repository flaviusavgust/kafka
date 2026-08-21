# kafka (local role)

Installs **Apache Kafka 4.x** (vanilla) in **KRaft combined mode** —
one process per node running `process.roles=broker,controller`. No ZooKeeper, no
TLS, no external repos. **SASL/SCRAM-SHA-512 authentication** on both listeners;
no authorizer, so any authenticated user has full access.

Steps: create `kafka` user + dirs → download/unpack the Apache tarball → render
`server.properties` (KRaft, SASL_PLAINTEXT/SCRAM, RF=3/min.insync=2,
`unclean.leader.election=false`, `auto.create.topics.enable=false`) →
`kafka-storage.sh format` once (guarded by `meta.properties`; bootstraps the
admin SCRAM credential via `--add-scram`) → systemd unit → start → create SCRAM
client users from `kafka_client_users`.

`node.id` and `controller.quorum.voters` come from the `kafka` inventory group
(`kafka_node_id` + `ansible_host` per host). The cluster id is `kafka_cluster_id`
(group_vars) — identical on all nodes.

Requires Java 17+ (installed by the playbook's `pre-tasks/java.yaml`) and the data
disk mounted at `/var/lib/kafka` (by the `manage-lvm` role) before this role runs.

## Authentication

Model: **let in anyone with valid credentials, then full access** (SASL/SCRAM
auth, no ACLs). Credentials come from group_vars (use ansible-vault for the
password values; avoid `"` in passwords — they end up inside JAAS strings):

```yaml
kafka_admin_user: admin                  # inter-broker/controller + admin CLI
kafka_admin_password: "{{ vault_kafka_admin_password }}"
kafka_client_users:
  - username: app-producer
    password: "{{ vault_kafka_app_producer_password }}"
```

The admin credential is bootstrapped at `kafka-storage.sh format --add-scram`
time (KIP-900) for the client/inter-broker listener. The **controller listener
uses SASL/PLAIN** with the same admin credential, statically in JAAS: SCRAM
cannot protect the controller quorum — controllers do not apply SCRAM records
from the bootstrap checkpoint until a leader is elected, so a multi-node quorum
with SCRAM on the controller listener deadlocks on mutual authentication
(verified empirically; identical bootstrap checkpoints on all nodes do not help).
Client users are created against the running cluster with `kafka-configs.sh`
(run_once, create-only): **adding** a user = add to the list and re-run the
playbook, no restart needed. **Password changes and deletions are not managed**
— do them manually, e.g.
`kafka-configs.sh ... --alter --entity-type users --entity-name <u> --delete-config SCRAM-SHA-512`,
then re-run the playbook to recreate with the new password.

Admin CLI ops on a node use `--command-config /opt/kafka/config/admin-client.properties`
(rendered by the role, mode 0600). A client config looks like:

```properties
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="app-producer" password="...";
```

Tasks whose command lines contain passwords run with `no_log` — set
`kafka_no_log: false` temporarily to debug them.

**Migrating an existing PLAINTEXT cluster:** the format guard means
`--add-scram` never runs on already-formatted storage, so the admin user won't
exist. This role assumes a fresh cluster: stop kafka, wipe
`/var/lib/kafka/data/*` on all nodes (data loss!), re-run the playbook.

**Topics are not created** (names not finalized). Create manually once agreed:

```bash
/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --command-config /opt/kafka/config/admin-client.properties --create \
  --topic <name> --partitions 3 --replication-factor 3 \
  --config min.insync.replicas=2
```

**Rolling restart:** a config change notifies a restart on all nodes. For a live
cluster restart one at a time (`--limit stage-did-kafka-0N`) and wait for
`--under-replicated-partitions` to clear between nodes.
