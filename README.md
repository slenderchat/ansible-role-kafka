# ansible-role-kafka

English | [Русский](README.ru.md)

An Ansible role that deploys and maintains an Apache Kafka cluster in KRaft mode
on RED OS 8 from the distribution RPM package. Every host of the `kafka`
inventory group runs a combined broker and controller.

Author: Nikita Grabar. Copyright (c) 2026 Nikita Grabar. All rights reserved.
See [COPYRIGHT](COPYRIGHT).

## Scope

The role:

- installs exactly `apache-kafka-3.9.1-1.red80`;
- writes `server.properties` and a KRaft systemd unit in place of the packaged
  ones;
- stops, disables and masks the packaged `zookeeper.service`;
- formats storage and writes an ownership record once, when it initializes a
  new cluster;
- on later runs, applies its configuration to a cluster it initialized and
  restarts nodes one at a time where needed.

The role does not:

- generate a cluster ID;
- adopt existing Kafka installations or data;
- repair partial initialization or recover hosts automatically;
- change the cluster ID, node IDs, controller membership or storage path of an
  initialized cluster, or replace nodes;
- upgrade Kafka to another version;
- configure TLS, authentication, ACLs, package repositories or firewalld.

## Requirements

### Managed hosts

- RED OS 8 with systemd. Hosts that report `ansible_distribution` as `RED` or
  `REDOS` with major version `8` are accepted.
- An IPv4 default route. The address of the default-route interface
  (`ansible_default_ipv4.address`) is used for listeners and controller quorum
  voters, and it must be unique within the group.
- Configured RED OS package repositories that provide
  `apache-kafka-3.9.1-1.red80` (in the tested environment it came from the
  `updates` repository). The role requests only this version-release and does
  not add repositories.
- Java is not installed explicitly. The package requires
  `java-11-openjdk-devel`, and dnf installs it as a dependency.
- TCP 9092 (broker) and 9093 (controller) open between all group members.
- Privilege escalation to root.

### Controller

- The role was verified with ansible-core 2.13.13 (Python 3.10, Jinja2 3.1.6)
  and the default `jinja2_native = False`. Other versions were not verified.
- Only `ansible.builtin` modules and filters are used, so no collections are
  required.

## Topology and connections

- `process.roles=broker,controller` on every host of the `kafka` group.
- `controller.quorum.voters` lists every group member as
  `<node_id>@<default IPv4>:9093`, ordered by node ID.
- Listeners: `PLAINTEXT://<default IPv4>:9092` for clients and brokers, and
  `CONTROLLER://<default IPv4>:9093` mapped to `PLAINTEXT`.
- All connections are unencrypted and unauthenticated. Restrict network access
  to these ports outside the role.
- Replication defaults depend on the group size *n*:
  - `default.replication.factor`, `offsets.topic.replication.factor` and
    `transaction.state.log.replication.factor` are min(3, n);
  - `min.insync.replicas` and `transaction.state.log.min.isr` are min(2, n).

## Variables

Two inventory variables are required. Neither has a default.

| Variable | Scope | Requirement |
|---|---|---|
| `kafka_cluster_id` | group | 22-character Kafka cluster ID (`^[A-Za-z0-9_-]{22}$`), identical on every host, compared case-sensitively |
| `kafka_node_id` | host | integer from 0 to 2147483647, unique within the group, stable for the life of the host |

Generate the cluster ID once, before Kafka is installed on the managed hosts.
For example, run this on the controller:

```sh
python3 -c 'import uuid,base64; print(base64.urlsafe_b64encode(uuid.uuid4().bytes).decode().rstrip("="))'
```

Alternatively, run `/opt/kafka/bin/kafka-storage.sh random-uuid` on any host
where the package is already installed.

Save the generated value as `kafka_cluster_id` and keep it after
initialization. Never change it or any node ID after initialization.
The role refuses such changes, and it also refuses a change of a host's
default-route IPv4 address, because that changes controller membership.

### Fixed constants

`vars/main.yml` defines fixed constants:

- package name and version-release;
- ports 9092 and 9093;
- `/opt/kafka` (home, `config/server.properties`, `logs`);
- `/etc/systemd/system/kafka.service`;
- `/var/lib/kafka` (state directory, the `data` storage directory and the
  ownership record `ansible-kafka-owner.json`).

Role `vars` take precedence over inventory variables, so inventory cannot change
these constants. Overriding them in any other way, for example with extra vars,
is not supported or tested.

Supported customization is limited to the inventory membership of the `kafka`
group and the two variables above. Configuration changes beyond these require
changes to the role itself.

## Installation and usage

Place the role in `roles/kafka/` of a parent Ansible project, so that the play
finds it by the name `kafka`:

```text
project/
├── inventory.yml
├── kafka.yml
└── roles/
    └── kafka/        # contents of this repository
```

For example, clone this repository into `roles/kafka`:

```sh
git clone https://github.com/slenderchat/ansible-role-kafka.git roles/kafka
```

Minimal inventory (`inventory.yml`):

```yaml
all:
  children:
    kafka:
      hosts:
        kafka-1.example.com:
          kafka_node_id: 1
        kafka-2.example.com:
          kafka_node_id: 2
        kafka-3.example.com:
          kafka_node_id: 3
      vars:
        kafka_cluster_id: "<22-character cluster ID>"
```

Playbook (`kafka.yml`). The role's error messages refer to it by this name.

```yaml
- name: Deploy Kafka cluster
  hosts: kafka
  become: true
  gather_facts: true
  strategy: linear
  any_errors_fatal: true
  roles:
    - role: kafka
      tags: [always]
```

Run it from the parent project directory:

```sh
ansible-playbook -i inventory.yml kafka.yml
```

### Execution restrictions

Preflight checks stop the play on every host before any change unless all of
the following hold:

- the play targets exactly the `kafka` group in one batch (no `--limit`, no
  `serial`);
- the run is not in check mode (`--check`);
- no `--tags` or `--skip-tags` is given;
- the platform requirements above are met;
- the variables are valid, node IDs and addresses are unique, and the cluster
  ID is identical on all hosts.

Some options are not detected:

- `--start-at-task` and `--step`;
- `--skip-tags always`, which skips the whole role without changes.

Do not use them.

## Behavior

### Run phases

Each run executes these phases on the whole group:

1. preflight;
2. read-only inspection;
3. group validation;
4. configuration;
5. initialization (new clusters only);
6. service start and restart.

The first three phases do not change hosts. With `strategy: linear` and
`any_errors_fatal: true`, a failure on any host stops the play before the
configuration phase.

### Inspection boundary

Inspection reads only the configured paths and services:

- the installed `apache-kafka` version, and `rpm -V` results for
  `/opt/kafka/config/server.properties` and
  `/etc/systemd/system/kafka.service`;
- the presence of these two files when the package is not installed;
- `kafka.service`: active state, enablement, drop-ins under
  `kafka.service.d`, main process start time and pending reload;
- `zookeeper.service`: active state, enablement and load state;
- running `kafka.Kafka` processes;
- the ownership record `/var/lib/kafka/ansible-kafka-owner.json`;
- `/var/lib/kafka/data`: existence, type, a complete listing, and
  `meta.properties`.

Kafka data in other locations, for example the package defaults under `/tmp`,
is not detected.

If the listing of an existing data directory is skipped or reports paths it
could not read, the play stops before any change.

### Host states and group decision

Each host is classified as one of four states:

- **clean**: no evidence that blocks initialization within the inspected scope.
  - The package is absent and no files remain at the configured paths, or the
    exact version is installed with both managed files unmodified.
  - `kafka.service` is not active and not enabled and has no Kafka drop-ins,
    and no Kafka process is running.
  - ZooKeeper is not active and not enabled.
  - No ownership record exists.
  - The data directory is absent or empty.
- **owned**: initialized by this role and consistent with the current
  variables.
  - The record matches: written by this role, with the same cluster ID, the
    same integer node ID, the same storage path and the same controller voters.
  - The exact package version is installed.
  - No Kafka drop-ins exist.
  - No Kafka server process is detected while `kafka.service` is not active.
    The check does not determine which service a process belongs to, and it
    does not detect an additional Kafka process while the service is active.
  - The data directory holds `meta.properties` with the same `cluster.id` and
    `node.id`.
- **inconsistent**: an ownership record exists, but the host deviates from it.
- **unowned**: Kafka traces without a record that establishes role ownership.

The group proceeds only in two cases:

- **all hosts clean**: initialize a new cluster;
- **all hosts owned**: apply the configuration.

Every other combination is refused before any change, with a per-host report.
Examples are a mix of clean and owned hosts, or any inconsistent or unowned
host.

### Initialization

When the whole group is clean, the role:

1. installs the package and writes the configuration;
2. runs `kafka-storage.sh format` with the supplied cluster ID on every host
   (without `--ignore-formatted`);
3. writes the ownership record on each host only after formatting succeeded
   there;
4. starts all nodes in one task;
5. runs readiness checks on the started nodes.

Initialization is not atomic across hosts.

### Restarts and readiness checks

A node that was running is restarted when one of these holds:

- this run changed `server.properties` or the unit;
- either file was modified after the Kafka main process started. Times are
  compared in whole microseconds, so a restart left pending by an interrupted
  run is not lost. A file whose modification time changed without a content
  change also triggers a restart.

Stopped nodes are started, not restarted.

When any node needs a restart:

1. every node first passes the readiness checks;
2. flagged nodes are restarted one at a time, in node ID order;
3. each restarted node passes the readiness checks before the next one
   restarts.

Before each restart, the role also refuses to continue if a group member has
already failed or become unreachable in the run.

Readiness checks run on a node from that node:

- its own ports 9093 and 9092 accept connections (up to 300 s);
- ports 9093 and 9092 of every group member are reachable (60 s per endpoint);
- `kafka-metadata-quorum.sh describe --replication` lists the node ID as
  Leader or Follower with lag 0;
- `kafka-broker-api-versions.sh` lists the node ID;
- `kafka-topics.sh --describe --under-replicated-partitions` returns nothing.

Readiness checks run only on nodes that were started in the run and around
rolling restarts. A run that starts and restarts nothing does not check
cluster health. The checks do not verify broker fencing state explicitly.

### Systemd units and ZooKeeper

The package installs its own `kafka.service` and `zookeeper.service` as regular
files in `/etc/systemd/system`. The packaged `kafka.service` requires
ZooKeeper, and a drop-in cannot remove that dependency.

The role therefore:

- replaces `/etc/systemd/system/kafka.service` with its own KRaft unit, so
  `rpm -V` reports the file as modified;
- loads the new unit, then stops and disables `zookeeper.service`;
- masks ZooKeeper by replacing its file with a link to `/dev/null`, because
  `systemctl mask` does not replace a regular file in `/etc/systemd/system`;
- runs `systemctl daemon-reload` when the mask changes, or when systemd still
  has a definition other than the mask loaded.

A stopped ZooKeeper may show as `failed`, because the packaged unit does not
treat exit code 143 as success. The role never unmasks ZooKeeper.

The following was previously observed when `apache-kafka-3.9.1-1.red80` was
reinstalled:

- the packaged `kafka.service` and `zookeeper.service` files are restored, and
  the mask is removed;
- `server.properties` is kept;
- a package scriptlet immediately restarts `kafka.service` with the packaged
  unit, which starts ZooKeeper.

This restart happens outside the role. If the exact supported package version
has to be reinstalled, do it on one node at a time, then run the role. The next
run is designed to restore the role's unit and the mask and to restart the
affected node. This reconciliation has not been repeated since the latest
change of the ZooKeeper reload condition (see "Verification").

### Recovery after an interrupted initial deployment

The manual recovery window starts when the first run writes
`server.properties`, not when it formats storage. If the first run stops before
every host has been formatted and has its ownership record, later runs classify
the affected hosts as unowned (or inconsistent) and refuse to continue.

The role does not repair this state. To retry, return every host to the clean
state described above:

- no ownership record and no content in `/var/lib/kafka/data`;
- `kafka.service` stopped and disabled, and no Kafka process running;
- `/opt/kafka/config/server.properties` and
  `/etc/systemd/system/kafka.service` either unmodified according to `rpm -V`,
  or absent together with the package.

Then run the playbook again. Do this only for a cluster that holds no data you
need, because it discards the storage.

## Verification

The current code was verified on a test group of three RED OS 8.0.3 hosts with
systemd 253, reporting `RED`. The controller ran
ansible-core 2.13.13 with `jinja2_native = False`.

Observed results:

- **Refusals before changes on a clean group.** The play stopped before any
  change when:
  - one host had a cluster ID that differed only in letter case;
  - a configuration file or a unit file remained without the package.
- **Initialization from clean hosts.**
  - A 3/3 quorum with zero lag formed, with three brokers.
  - A topic with three partitions, replication factor 3 and
    `min.insync.replicas=2` accepted messages with `acks=all`, and all
    messages were read back.
- **Unchanged runs.** Repeated runs reported `changed=0`, with no formatting,
  record rewrite, daemon reload or restart.
- **Start-time precision.**
  - A file modified earlier within the same second as the process start did
    not trigger a restart.
  - A file modified later within the same second did, and only on that node.
- **Interrupted run.** After a run stopped once the configuration was written,
  the next run restarted all nodes one at a time in node ID order.
- **Refusals on an initialized cluster.** The play stopped before any change
  in these cases:
  - a boolean or float node ID in the ownership record (inconsistent);
  - a removed ownership record (unowned);
  - a missing or contradictory `meta.properties` (inconsistent);
  - one clean host alongside owned hosts.
- **Pending ZooKeeper mask.** ZooKeeper was running from the packaged unit,
  and its file had been replaced with the mask without a reload. The role
  stopped ZooKeeper, reloaded systemd, and ZooKeeper became masked. Kafka was
  not restarted.

Not verified at runtime:

- **Incomplete data directory listing.** This fault-injection case was not
  executed. In the test environment, root read the directory without error.
  The stop condition relies on the documented behavior of `ansible.builtin.find`
  in ansible-core 2.13.13, which was confirmed by reading the module source,
  not by a run.
- **Package reinstallation.** Reinstalling the exact package version and the
  following reconciliation were exercised before the latest change of the
  ZooKeeper reload condition. The reconciliation has not been repeated since.
- **Other environments and failures:**
  - hosts reporting `REDOS`;
  - group sizes other than three;
  - other ansible-core versions;
  - an active firewalld;
  - loss of a node during the rolling restart loop.

## Copyright

Copyright (c) 2026 Nikita Grabar. All rights reserved. See [COPYRIGHT](COPYRIGHT).
