# stratum-proxy

Java stratum proxy for crypto-currency mining. Proxies Stratum protocol connections between miners and pools.

## Project Layout

```
src/main/java/strat/mining/stratum/proxy/
  database/         - DatabaseManager (db4o-backed hashrate history)
  manager/          - HashrateRecorder, ProxyManager
  configuration/    - ConfigurationManager, model/Configuration
  rest/             - ProxyResources (REST API via Grizzly/Jersey)
  utils/            - Timer (custom scheduler using cachedThreadPool)
  Launcher.java     - Entry point
lib/                - Local Maven repo (includes db4o 8.0.249)
assemblies/         - Maven assembly descriptors
```

## Build

```bash
mvn clean package
# Output jar: target/stratum-proxy-0.8.1.jar
```

## Deploy

Proxy instances live at `/home/mine/proxs/stratum-proxy-*/stratum-proxy.jar` (17 instances).
Each instance is a systemd service named `stratum-proxy-<name>`.

Copy built jar to all instances:
```bash
for dir in /home/mine/proxs/stratum-proxy-*/; do
  cp target/stratum-proxy-0.8.1.jar "${dir}stratum-proxy.jar"
done
```

Restart all services:
```bash
for svc in $(systemctl list-units --type=service --all --no-legend | grep "stratum-proxy" | awk '{print $1}'); do
  sudo systemctl restart "$svc"; sleep 3
done
```

## Key Components

- **DatabaseManager** - Singleton wrapping two db4o `ObjectContainer`s: `dbpools` and `dbusers`. Stores hashrate samples. db4o never reclaims space after deletes, causing the file (and Java memory footprint) to grow indefinitely.
- **HashrateRecorder** - Samples pool/user hashrate every `hashrateDatabaseSamplingPeriod` seconds (default 60). Deletes records older than `hashrateDatabaseHistoryDepth` days (default 7, user configs use 3). Uses the `Timer` utility which runs tasks on a `cachedThreadPool`.
- **Timer** - Custom scheduler. Tasks run concurrently on pooled threads, so `DatabaseManager` methods need synchronization.

## Configuration (per instance, stratum-proxy.conf)

| Key | Default | Notes |
|-|-|-|
| `databaseDirectory` | `<install>/database/` | Path to db4o files |
| `hashrateDatabaseSamplingPeriod` | 60 | Seconds between captures |
| `hashrateDatabaseHistoryDepth` | 7 | Days of history to retain |

## Backlog

- Add periodic db4o defragmentation to prevent memory growth (see `docs/superpowers/plans/2026-06-09-db4o-defrag.md`)

## Session Log

### Session 1 - 2026-06-09
- Diagnosed memory growth: db4o never reclaims space after record deletion
- Decided on periodic defrag via `DatabaseManager.defragment()` + daily timer in `HashrateRecorder`
- Created CLAUDE.md and implementation plan
