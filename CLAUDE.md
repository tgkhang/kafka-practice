# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal **Kafka learning playground**, not a single deployable app. It collects independent, self-contained pieces that follow the Conduktor "Kafka for Beginners" course: CLI cheat-sheet scripts, Docker Compose stacks for running a local cluster, and several standalone Java demo programs. Nothing here is wired together — each top-level folder stands alone.

## Layout (each is independent)

- `1-kafka-cli/`, `2-kafka-extended/`, `3-kafka-topic-configurations/` — `.sh` files that are **command reference sheets**, not runnable scripts. Lines are `kafka-topics.sh ...` / `kafka-console-producer.sh ...` invocations meant to be copy-pasted one at a time against a running broker at `localhost:9092`. The intended workflow (see `1-kafka-cli/README.md`) is to run them **inside the `docker/base` broker container**: `docker exec -it kafka bash`, then `export PATH=$PATH:/opt/kafka/bin` (the `apache/kafka` image keeps the tools at `/opt/kafka/bin/*.sh`, not on PATH).
- `2-kafka-extended/` — Kafka Connect (Wikimedia source, Elasticsearch sink) and Schema Registry samples; `config/*.properties` are connector/worker configs.
- `docker/base/` — **the primary, minimal learning cluster**: a single `apache/kafka` broker (KRaft mode, no ZooKeeper, no UI, no DB). This is what the `1-kafka-cli` lessons target. Up/down: `docker compose -f docker/base/docker-compose.yml up -d` / `down` (`-v` to wipe data). Single-broker constraints: replication factor is capped at 1; `KAFKA_NUM_PARTITIONS` (default 1) controls auto-created topics. See `docker/base/README.md`.
- `docker/conduktor/` (`full-stack.yml`, `kafka-simple.yml`) — heavier Docker Compose stacks that add the **Conduktor web UI + Postgres**. Optional; only when you want a GUI. **Note:** git history shows these `.yml` files were moved from the repo root into `docker/conduktor/`; the root `README.md` still references the old root paths (`docker compose -f full-stack.yml up`) — use the `docker/conduktor/` path instead.
- `kafka-beginners-course/` — a **Gradle multi-module** project (the main body of Java code).
- `kafka-java/` — a separate, **Maven** scratch project (`com.xxxx`, just a `Main` stub), unrelated to the Gradle project. Targets Java 25.

## kafka-beginners-course (Gradle)

Multi-module build; modules declared in `settings.gradle`: `kafka-basics`, `kafka-producer-wikimedia`, `kafka-consumer-opensearch`, `kafka-streams-wikimedia`. Uses `kafka-clients` / `kafka-streams` **3.3.1**.

Run Gradle from the `kafka-beginners-course/` directory (use the wrapper):

```bash
./gradlew build                 # compile all modules
./gradlew :kafka-basics:build   # build one module
./gradlew test                  # JUnit 5 (useJUnitPlatform); no real tests exist yet
```

There is **no `application` plugin and no `mainClass`** configured. Each demo is a class with its own `public static void main` and is intended to be **run individually from the IDE** (IntelliJ), not via `gradle run`. To run one from the CLI you must build and invoke it on the classpath manually.

Key demos:
- `kafka-basics/` — progressively richer producer/consumer examples (`ProducerDemo`, `...WithCallback`, `...Keys`; `ConsumerDemo`, `...WithShutdown`, `...Cooperative`, plus `advanced/` rebalance-listener and threaded variants).
- `kafka-producer-wikimedia/` — produces the Wikimedia SSE stream into Kafka.
- `kafka-consumer-opensearch/` — consumes into OpenSearch (its own `docker-compose.yml` for OpenSearch).
- `kafka-streams-wikimedia/` — Kafka Streams topology (`processor/` builders).

## kafka-java (Maven)

```bash
cd kafka-java
mvn compile
mvn exec:java -Dexec.mainClass=com.xxxx.Main   # if exec plugin is added; not configured yet
```

## Conventions

- Broker address in demo code is hardcoded, sometimes `127.0.0.1:9092` and sometimes `localhost:9092` — match the surrounding file rather than normalizing.
- Logging is SLF4J + `slf4j-simple` across the Gradle modules.
- Bring up a cluster first (`docker/conduktor/*.yml`) before running any CLI script or Java demo; Conduktor UI is at `localhost:8080`, broker at `localhost:9092`.
