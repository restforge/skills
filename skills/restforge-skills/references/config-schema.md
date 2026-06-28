# Reference: Config Schema (db-connection.env)

> **Offline mirror.** This file mirrors `setup_get_config_schema` from the
> installed RESTForge platform. The live tool is authoritative — when this file
> and the tool disagree, trust the tool, then update this file. Parameter set and
> defaults can change between platform versions.

Source: `setup_get_config_schema` — installed platform version.
Total: 63 parameters. Required parameters marked **\***.
Schema version: 1.0.

---

## Section: License

| Parameter | Type | Default | Required |
|---|---|---|---|
| `LICENSE` | string | `XXXX-XXXX-XXXX-XXXX` | **\*** |

---

## Section: Server

| Parameter | Type | Default | Required |
|---|---|---|---|
| `SERVER_ADDRESS` | string | `127.0.0.1` | **\*** |
| `SERVER_PORT` | integer | `3000` | **\*** |

---

## Section: Live Sync (WebSocket)

LIVE_SYNC_ENABLED=true requires an API Key (`KEY=...`) for WebSocket client authentication.

| Parameter | Type | Default |
|---|---|---|
| `LIVE_SYNC_ENABLED` | boolean | `false` |
| `LIVE_SYNC_PORT` | integer | `3033` |

---

## Section: Redis

| Parameter | Type | Default |
|---|---|---|
| `REDIS_HOST` | string | `localhost` |
| `REDIS_PORT` | integer | `6380` |
| `REDIS_PASSWORD` | string | `` |
| `REDIS_DB` | integer | `0` |

---

## Section: Export

| Parameter | Type | Default |
|---|---|---|
| `EXPORT_FILE_EXPIRY` | integer | `3600000` |
| `EXPORT_CHUNK_SIZE` | integer | `1000` |

---

## Section: Kafka

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `KAFKA_ENABLED` | boolean | `false` | |
| `KAFKA_BROKERS` | string | `localhost:9092` | Comma-separated for multiple brokers |
| `KAFKA_CONNECTION_TIMEOUT` | integer | `3000` | |
| `KAFKA_REQUEST_TIMEOUT` | integer | `25000` | |
| `KAFKA_TOPIC_PATTERN` | string | `{module}.{endpoint}.events` | |
| `KAFKA_TENANT_ID` | string | `default` | |
| `KAFKA_SESSION_TIMEOUT` | integer | `30000` | |
| `KAFKA_HEARTBEAT_INTERVAL` | integer | `3000` | |
| `KAFKA_MAX_BYTES_PER_PARTITION` | integer | `1048576` | |
| `KAFKA_AUTO_COMMIT` | boolean | `false` | |
| `KAFKA_AUTO_COMMIT_INTERVAL` | integer | `5000` | |
| `KAFKA_RETRY_ATTEMPTS` | integer | `3` | |
| `KAFKA_RETRY_DELAY` | integer | `1000` | |
| `KAFKA_RETRY_MAX_DELAY` | integer | `30000` | |
| `KAFKA_SSL` | boolean | `false` | |
| `KAFKA_LOG_LEVEL` | string | `info` | |

SASL Authentication (optional, uncomment when broker requires authentication):
`KAFKA_SASL_MECHANISM` (plain/scram-sha-256/scram-sha-512), `KAFKA_SASL_USERNAME`,
`KAFKA_SASL_PASSWORD`.

---

## Section: Database

| Parameter | Type | Default | Required |
|---|---|---|---|
| `DB_TYPE` | string | `postgresql` | **\*** — postgresql, mysql, oracle, sqlite |
| `DB_HOST` | string | `127.0.0.1` | **\*** |
| `DB_PORT` | integer | `5432` | **\*** |
| `DB_USER` | string | `postgres` | **\*** |
| `DB_PASSWORD` | string | `your_password_here` | **\*** |
| `DB_NAME` | string | `your_database_name` | **\*** |

For SQLite: set `DB_TYPE=sqlite` and `DB_NAME=./data/myapp.db`.
`DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` are ignored for SQLite.

---

## Section: Logging

| Parameter | Type | Default |
|---|---|---|
| `LOG_LEVEL` | string | `debug` |
| `LOG_TO_FILE` | boolean | `true` |

---

## Section: SQL Logging

| Parameter | Type | Default |
|---|---|---|
| `SQL_LOG_ENABLED` | boolean | `false` |
| `SQL_LOG_LEVEL` | string | `debug` |
| `SQL_LOG_PARAMS` | boolean | `false` |
| `SQL_LOG_SLOW_THRESHOLD` | integer | `1000` |

---

## Section: Cache

| Parameter | Type | Default |
|---|---|---|
| `CACHE_ENABLED` | boolean | `false` |
| `CACHE_TTL` | integer | `300` |

---

## Section: Job Scheduler

| Parameter | Type | Default |
|---|---|---|
| `JOB_ENABLED` | boolean | `false` |
| `JOB_CONCURRENCY` | integer | `5` |
| `JOB_RETENTION_HOURS` | integer | `72` |
| `JOB_FAILED_RETENTION_HOURS` | integer | `168` |
| `JOB_SHUTDOWN_TIMEOUT` | integer | `10000` |
| `JOB_STALLED_INTERVAL` | integer | `30000` |
| `JOB_MAX_STALLED_COUNT` | integer | `2` |

---

## Section: Distributed Lock

| Parameter | Type | Default |
|---|---|---|
| `LOCK_DISTRIBUTED_ENABLED` | boolean | `false` |
| `LOCK_DISTRIBUTED_TTL` | integer | `10` |
| `LOCK_RESOURCE_MAX_TTL` | integer | `600` |
| `LOCK_DISTRIBUTED_RETRY` | integer | `3` |
| `LOCK_DISTRIBUTED_RETRY_DELAY` | integer | `100` |
| `LOCK_DISTRIBUTED_STRATEGY` | string | `reject` |

---

## Section: ID Generator

| Parameter | Type | Default |
|---|---|---|
| `IDGEN_ENABLED` | boolean | `false` |
| `IDGEN_IDEM_TTL` | integer | `600` |
| `IDGEN_COUNTER_TTL_MONTHLY` | integer | `2764800` |
| `IDGEN_COUNTER_TTL_DAILY` | integer | `172800` |
| `IDGEN_DEFAULT_MAX_RETRY` | integer | `10` |
| `IDGEN_DEFAULT_PIN_DIGITS` | integer | `6` |
| `IDGEN_DEFAULT_SERIAL_PATTERN` | string | `XXXX-XXXX-XXXX-XXXX` |
| `IDGEN_DEFAULT_CODE_PATTERN` | string | `9999-9999` |
| `IDGEN_ALLOW_RESET` | boolean | `false` |
