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

Requires Java 11 JDK to compile (temurin-11-jdk-amd64). Proxies must RUN on Java 8 JRE (temurin-8-jre-amd64) — Jersey 2.14 has runtime errors on Java 11.

```bash
JAVA_HOME=/usr/lib/jvm/temurin-11-jdk-amd64 mvn clean package -q
# Output jar: target/stratum-proxy-0.8.1.jar
```

To set default java back to Java 8 after installing temurin-11-jdk:
```bash
sudo update-alternatives --set java /usr/lib/jvm/temurin-8-jre-amd64/bin/java
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

- Fix partial defrag logging: split try/catch per-database in `defragment()` so failures identify which file failed (`DatabaseManager.java:~106`)
- Merge `feature/db4o-defrag` to master after above fix
- Set system default java back to Java 8: `sudo update-alternatives --set java /usr/lib/jvm/temurin-8-jre-amd64/bin/java`, then restart all 17 proxy services

## Session Log

### Session 2 - next (Session 2)

### Session 1 - 2026-06-09 (complete)

- Diagnosed memory growth: db4o never reclaims space after record deletion
- Decided on periodic defrag via `DatabaseManager.defragment()` + daily timer in `HashrateRecorder`
- Implemented all 4 tasks on branch `feature/db4o-defrag`: synchronized DatabaseManager, added defragment(), daily HashrateRecorder timer, built and deployed jar to all 17 instances
- Discovered installing temurin-11-jdk changed system default java to 11 — proxies run but show Jersey HK2 warnings; need to switch back to Java 8 JRE
- Two items remain before merge: partial defrag logging fix + java default fix + final restart
