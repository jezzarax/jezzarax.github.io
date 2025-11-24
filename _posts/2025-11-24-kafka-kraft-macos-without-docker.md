---
layout: post
title: Kafka KRaft on macOS without Docker
---

Running docker on apple silicon is still somehow a struggle, here are some notes about how to set native kafka with KRaft on macos docker-free and suitable for development.

<!--more-->

## Install & baseline
- `brew install kafka`
- Homebrew puts configs in `/opt/homebrew/etc/kafka/` and binaries in `/opt/homebrew/Cellar/kafka/<version>/bin`.

## Switch the config to KRaft with custom ports
Replace the default ZooKeeper config with the KRaft template and move off the default ports (I used 19092/19093 to keep 9092/9093 free):

```bash
sudo cp /opt/homebrew/Cellar/kafka/4.1.1/libexec/config/kraft/server.properties \
  /opt/homebrew/etc/kafka/server.properties
```

Open `/opt/homebrew/etc/kafka/server.properties` and set these lines (create them if missing):
- `listeners=PLAINTEXT://:19092,CONTROLLER://:19093`
- `advertised.listeners=PLAINTEXT://localhost:19092,CONTROLLER://localhost:19093`
- `controller.quorum.voters=1@localhost:19093`
- `log.dirs=/opt/homebrew/var/lib/kraft-dev-logs`

## Format the storage directory
KRaft needs a formatted metadata log. Clear any old data (I had it from previous installation, and it was not removed with `brew uninstall kafka` for whatever reason) and format with a new cluster id:

```bash
sudo rm -rf /opt/homebrew/var/lib/kraft-dev-logs
sudo mkdir -p /opt/homebrew/var/lib/kraft-dev-logs
CLUSTER_ID=$(/opt/homebrew/Cellar/kafka/4.1.1/bin/kafka-storage random-uuid)
sudo /opt/homebrew/Cellar/kafka/4.1.1/bin/kafka-storage format \
  --cluster-id "$CLUSTER_ID" \
  --config /opt/homebrew/etc/kafka/server.properties
```

## Start and verify (manual run)
Launch manually to keep control of ports and logs:

```bash
kafka-server-start /opt/homebrew/etc/kafka/server.properties
```

Then, in another shell, create a topic and produce/consume:

```bash
kafka-topics --bootstrap-server localhost:19092 --create --topic test
kafka-console-producer --broker-list localhost:19092 --topic test
kafka-console-consumer --bootstrap-server localhost:19092 --topic test --from-beginning
```

## Notes
- No ZooKeeper is involved; 4.1 ships KRaft-only by default, extra complexity (especially for local dev setup) is gone.
- If you change ports again, update `listeners`, `advertised.listeners`, and the port inside `controller.quorum.voters` together, then rerun the format step.
- For local dev you can skip TLS/SASL; defaults are PLAINTEXT.

Obvious disclaimer, don't use it in production :)
