# docker/base — a minimal Kafka for CLI learning

A single-container Apache Kafka broker, stripped down to the essentials so you
can learn Kafka **from the command line** without the Conduktor web UI, a
database, or ZooKeeper getting in the way.

What you get:

- **1 image only**: `apache/kafka` (the official Apache image).
- **KRaft mode** — Kafka manages its own metadata, so there is **no ZooKeeper**.
- The single broker plays **both roles**: `broker` (serves clients) and
  `controller` (manages cluster metadata).
- All the `kafka-*.sh` CLI tools ship *inside* this container, so you don't have
  to install Kafka on your machine.

---

## 1. Start / stop the broker

Run these from the **repo root** (`kafka learnnninng/`).

```powershell
# Start (in the background)
docker compose -f docker/base/docker-compose.yml up -d

# Check it's healthy (wait until STATUS shows "healthy")
docker compose -f docker/base/docker-compose.yml ps

# Follow the logs
docker compose -f docker/base/docker-compose.yml logs -f kafka

# Stop, keeping your topics/data
docker compose -f docker/base/docker-compose.yml down

# Stop AND erase all data (fresh start next time)
docker compose -f docker/base/docker-compose.yml down -v
```

---

## 2. Open a shell inside the broker (Windows)

The Kafka CLI tools ship **inside the container** at `/opt/kafka/bin/`.
They are not installed on your Windows machine.

**Step 1 — open PowerShell** (Win + X → "Terminal" or search for PowerShell).

**Step 2 — enter the container:**

```powershell
docker exec -it kafka bash
```

Your prompt changes to something like `root@kafka:/#` — you are now inside the
Linux container, not Windows.

**Step 3 — add the tools to your PATH (once per shell session):**

```bash
export PATH=$PATH:/opt/kafka/bin
```

**Step 4 — run any CLI command, e.g.:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Step 5 — when you are done, leave the container:**

```bash
exit
```

The broker keeps running after you exit. Your shell goes back to the Windows
PowerShell prompt.

---

### Need a second terminal? (producer in one, consumer in another)

Open a **new PowerShell window** and repeat steps 2–3 in it. Each window gives
you an independent shell inside the same running broker. This is how the
consumer-group lessons (files 3 and 5 in `1-kafka-cli/`) work.

---

### One-liner (no interactive shell)

If you just want to run a single command without entering the container:

```powershell
docker exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

## 3. How clients connect

| Who is connecting                         | Bootstrap server to use |
|-------------------------------------------|-------------------------|
| CLI **inside** the `kafka` container      | `localhost:9092`        |
| Java demos / apps **on your host**        | `localhost:9092`        |

Port `9092` is published to your host, and the broker *advertises* itself as
`localhost:9092`, so the same address works in both places. This is also why the
Java code under `kafka-beginners-course/` (which points at `localhost:9092` /
`127.0.0.1:9092`) talks to this broker with no changes.

---

## 4. Replication factor — and when you need the multi-broker setup

**Replication factor** is how many brokers hold a copy of each partition.
RF=1 means one copy (no redundancy). RF=3 means three copies — if one broker
dies, two others still have the data.

With only one broker here, `--replication-factor` can only ever be `1`.
Trying `--replication-factor 2` will fail with:

```text
Replication factor: 2 larger than available brokers: 1
```

This is not a misconfiguration — it is physically impossible to store 2 copies
when there is only 1 broker.

**To practice replication factor > 1**, use the 3-broker setup instead:

```powershell
# Stop this single-broker first (both use port 9092)
docker compose -f docker/base/docker-compose.yml down

# Start the 3-broker cluster
docker compose -f docker/multi-broker/docker-compose.yml up -d
```

See [`../multi-broker/README.md`](../multi-broker/README.md) for what to do
there — it explains Leader, Replicas, ISR, and has a broker-failure exercise.

---

### What IS useful to practice here first

- **`KAFKA_NUM_PARTITIONS: 1`** — auto-created topics start with 1 partition.
  The CLI lessons use this deliberately: you create a topic without
  `--partitions` (gets 1 partition), then create another with `--partitions 3`
  and see the difference in `--describe` output. Change this value in
  [`docker-compose.yml`](docker-compose.yml) and do `down -v` + `up` to
  reset if you want to experiment with a different default.

---

## 5. Common issues

- **`Connection refused` right after `up`** — the broker needs a few seconds.
  Wait for `ps` to report `healthy`.
- **Want a clean slate?** `down -v` deletes the `kafka_data` volume, wiping all
  topics and messages.
- **Port 9092 already in use** — something else (another Kafka, an old
  container) holds the port. Stop it, or change the `9092:9092` mapping.
