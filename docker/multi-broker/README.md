# docker/multi-broker — 3-broker cluster for practising replication

Use this when you want to practice **replication factor > 1**.  
For basic topic/producer/consumer CLI work, `../base/` is simpler.

---

## What replication factor means

Every partition of a topic is stored on N brokers, where N = replication factor.

```txt
Topic "orders", 3 partitions, RF = 3

  Partition 0 → stored on broker 1 (leader), broker 2, broker 3
  Partition 1 → stored on broker 2 (leader), broker 3, broker 1
  Partition 2 → stored on broker 3 (leader), broker 1, broker 2
```

- The **leader** handles all reads and writes for that partition.
- The others are **replicas** (followers) — they copy data from the leader.
- If the leader dies, Kafka promotes a replica automatically.
- `--replication-factor 2` means any 2 of the 3 brokers hold the data.
  You can lose 1 broker and still read/write.
- `--replication-factor 3` means all 3 hold it.
  You can lose 2 brokers and still survive (1 ISR left).

With only 1 broker (`../base/`) there is nowhere to put a second copy, so
`--replication-factor 2` fails. Here you have 3 brokers, so 1, 2, or 3 all work.

---

## Start / stop

Run from the **repo root** (`kafka learnnninng/`).

```powershell
# Start all 3 brokers
docker compose -f docker/multi-broker/docker-compose.yml up -d

# Check all three show "Up"
docker compose -f docker/multi-broker/docker-compose.yml ps

# Stop, keeping data
docker compose -f docker/multi-broker/docker-compose.yml down

# Stop and wipe all data
docker compose -f docker/multi-broker/docker-compose.yml down -v
```

> **Port conflict:** `../base/` and this setup both use port 9092.
> Stop one before starting the other.

---

## Open a shell for CLI practice

Exec into **kafka1** (the same steps as `../base/README.md`):

```powershell
docker exec -it kafka1 bash
```

```bash
export PATH=$PATH:/opt/kafka/bin
```

Use `--bootstrap-server localhost:9092` for all commands — you're talking to
kafka1, which knows about all 3 brokers and will route things correctly.

---

## Things to try

**Create a topic with RF=3 (all brokers hold every partition):**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic my-replicated-topic \
  --partitions 3 --replication-factor 3
```

**Describe it — notice Leader, Replicas, Isr on each partition:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic my-replicated-topic
```

Output looks like:

```
Partition: 0   Leader: 3   Replicas: 3,1,2   Isr: 3,1,2
Partition: 1   Leader: 1   Replicas: 1,2,3   Isr: 1,2,3
Partition: 2   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
```

- **Leader** — which broker handles reads/writes for this partition right now.
- **Replicas** — all brokers that hold a copy (always equals RF).
- **Isr** (In-Sync Replicas) — replicas fully caught up with the leader.
  Healthy = Isr count equals Replicas count.

**Simulate a broker failure:**

Open a second PowerShell window and kill kafka2:

```powershell
docker stop kafka2
```

Back in the kafka1 shell, describe the topic again:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic my-replicated-topic
```

You will see Isr shrink (kafka2 drops out) and partitions whose leader was
kafka2 will elect a new leader from the remaining brokers. The topic is still
fully readable and writable.

Bring kafka2 back:

```powershell
docker start kafka2
```

Describe again — kafka2 rejoins Isr as it catches up.

---

## ISR vs Replicas

| Replicas | Isr | What it means |
|----------|-----|---------------|
| `1,2,3`  | `1,2,3` | All healthy |
| `1,2,3`  | `1,3`   | Broker 2 is lagging or down |
| `1,2,3`  | `1`     | Only 1 broker left; risky |

`min.insync.replicas` (set to 2 in this compose file) means a producer with
`acks=all` requires at least 2 ISR before the write is acknowledged. Drop below
that and writes with `acks=all` will be rejected — a safety net against data
loss.
