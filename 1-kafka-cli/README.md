# 1 — Kafka CLI basics

Hands-on tour of the core Kafka command-line tools, using the minimal broker in
[`../docker/base/`](../docker/base/). No web UI — just you and the CLI.

## Before you start

1. **Start the broker** (from the repo root):

   ```powershell
   docker compose -f docker/base/docker-compose.yml up -d
   ```

2. **Open a shell inside the broker** and put the tools on your `PATH`:

   ```powershell
   docker exec -it kafka bash
   ```

   ```bash
   export PATH=$PATH:/opt/kafka/bin
   ```

Now the commands in the `.sh` files below run as written. The broker is always
`--bootstrap-server localhost:9092`.

> These `.sh` files are **cheat-sheets, not scripts** — copy/paste and run the
> lines one at a time, observing what each does. Don't execute a whole file.

## The lessons (in order)

| File | What you learn |
| ---- | -------------- |
| [`0-kafka-topics.md`](0-kafka-topics.md) | Create, list, describe, delete topics; partitions & replication |
| [`1-kafka-console-producer.md`](1-kafka-console-producer.md) | Produce messages; `acks`; keys; auto-topic-creation |
| [`2-kafka-console-consumer.md`](2-kafka-console-consumer.md) | Consume; `--from-beginning`; print keys/timestamps/partitions |
| [`3-kafka-console-consumer-in-groups.md`](3-kafka-console-consumer-in-groups.md) | Consumer groups; how partitions are shared across consumers |
| [`4-kafka-consumer-groups.md`](4-kafka-consumer-groups.md) | Inspect groups, lag, and offsets |
| [`5-reset-offsets.md`](5-reset-offsets.md) | Replay data by resetting a group's offsets |

## Tip: multiple terminals

Several lessons need a producer in one terminal and a consumer in another. Just
open another PowerShell window and run `docker exec -it kafka bash` again
(remember the `export PATH` line in each new shell).
