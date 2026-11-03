# Project context

Durable context for agents. Maintenance rules are in `AGENTS.md`. User-facing
behavior is documented in `README.md`; this file records only what is not
obvious from it or from the code.

## Purpose

A standalone Ansible role that deploys a KRaft Kafka cluster, with every host
running a combined broker and controller over PLAINTEXT. It targets
`apache-kafka-3.9.1-1.red80` on RED OS 8 and is used from a parent project as
`roles/kafka/`.

## State

- The role code is complete for its documented scope. No changes are planned.
- Verification evidence and its limits are summarized in the "Verification"
  section of `README.md`.
- Open verification gaps:
  - the incomplete-listing stop condition was confirmed only from the
    `ansible.builtin.find` source in ansible-core 2.13.13, not by a run;
  - reconciliation after a package reinstall has not been repeated since the
    ZooKeeper reload condition switched to `LoadState`;
  - hosts reporting `REDOS`;
  - group sizes other than three;
  - an active firewalld;
  - loss of a node during the rolling restart loop.

## Decisions and constraints

- **Interface.** The only inputs are `kafka_cluster_id` (group) and
  `kafka_node_id` (host). Everything else is a fixed constant in `vars/`, which
  takes precedence over inventory. Do not move constants to `defaults/` or add
  variables without an explicit decision.
- **Group decision.** The group is either all clean (initialize) or all owned
  (apply). Everything else is refused before any change. There is no adoption,
  repair or automatic recovery, and initialization is not atomic across hosts.
- **Inspection boundary.** Inspection covers only the configured paths (config
  file, unit file, data directory, ownership record) and the two services.
  Recorded `log_dirs` are compared with the configured path but never opened.
- **Systemd unit.** The package installs its `kafka.service` (which
  `Requires=zookeeper.service`) at the administrative path
  `/etc/systemd/system`. A drop-in cannot remove a dependency, so the role
  replaces that file in place.
- **ZooKeeper mask.** The packaged `zookeeper.service` is a regular file in
  `/etc/systemd/system`, where `systemctl mask` refuses to act. The role
  replaces the file with a `/dev/null` link after stopping and disabling the
  service; stopping it earlier would stop Kafka through `Requires=`.
- **ZooKeeper reload signal.** The reload after masking runs when the mask
  changed or `LoadState` is not `masked`. `NeedDaemonReload` is unusable here
  on systemd 253 (observed):
  - a unit linked to `/dev/null` always reports `yes`;
  - a running unit whose file was replaced without a reload reports `no` with
    `LoadState=loaded`;
  - an inactive unit is re-read from disk on demand and already shows
    `masked`.
- **Restart detection.** The start time is read from
  `ExecMainStartTimestamp` with `--timestamp=us+utc` and `LC_ALL=C`, and
  converted to integer microseconds without floats or time zones. It is
  compared with `stat` mtime × 10^6, truncated. Whole-second timestamps caused
  wrong decisions within the same second.
- **Ansible 2.13 with `jinja2_native = False`:**
  - numbers produced by `set_fact` are stored as strings, so compare them
    with `| int`;
  - `unique` is case-insensitive by default, so cluster IDs use
    `unique(case_sensitive=true)`;
  - a `None` computed as a standalone task variable stops being `none`, so the
    existence of `meta.properties` is kept as a separate boolean fact.

  Do not enable native mode to work around type errors.
- **Formatting.** The cluster ID is passed as a single `--cluster-id=<id>`
  argument in `argv`, so a valid ID that starts with `-` is not parsed as an
  option.
- **Ownership record.** The record is written only after formatting succeeds,
  with `force: false`. Its `node_id` must pass the `integer` test, which
  rejects booleans and floats.
- **Package facts (3.9.1-1.red80, observed):**
  - `config/server.properties` is `%config(noreplace)`;
  - the unit files are not `%config`;
  - reinstalling restores both unit files and immediately restarts
    `kafka.service` through `systemd-update-helper`;
  - the JVM exits with 143 on SIGTERM.
- **Unsupported options.** `--start-at-task`, `--step` and
  `--skip-tags always` are not detected by preflight.

## Blockers

- None.

## Next steps

- None required. Closing the verification gaps above needs a disposable
  RED OS 8 test group.

## Useful commands

- Syntax check from a parent project that holds this role in `roles/kafka/`,
  with ansible-core 2.13.13:
  `ansible-playbook -i inventory.yml --syntax-check kafka.yml`
