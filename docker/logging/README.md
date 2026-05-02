# Container log shipping

This stack does **not** run a log shipper as part of `docker compose`.
Container `stdout`/`stderr` shipping into RabbitMQ is handled by an
out-of-stack daemon, **`karotten`**, which runs on the host
and reads `/var/run/docker.sock`.

Source, install, and operator docs:
<https://github.com/paranoid-software/karotten>

This document describes the **contract** (what subscribers can rely on
about the messages that arrive in RabbitMQ) so that consumers in this
stack — or anywhere on `minion` — can be written without reading the
shipper's source.

## Contract

- **Broker**: in-stack RabbitMQ (`the-rabbit`).
- **Vhost**: `minion`.
- **Exchange**: `containers-logs` (topic, durable).
- **Routing key**: always **three segments**, all derived from
  Docker-level metadata only — the shipper never parses message bodies.

```
<project>.<service>.<stream>
```

| Segment | Source | Notes |
|---|---|---|
| `project` | `com.docker.compose.project` label of the container | `standalone` for non-compose containers. Any literal `.` becomes `-`. |
| `service` | `com.docker.compose.service` label of the container | Falls back to the Docker container name for non-compose containers. Any literal `.` becomes `-`. |
| `stream` | Docker stdio stream | Either `stdout` or `stderr`. |

Topic-exchange wildcards work as usual: `*` matches one segment, `#`
matches zero or more. So you can subscribe to "any stderr from the
looky stack" with `my-looky.*.stderr`, "everything from one service"
with `*.<service>.#`, and so on.

Replicas of a service share the routing key. To differentiate them,
read `container_id` / `container_name` from the message body.

## Message body

Each message is a JSON envelope. Fields:

| Field | Type | Description |
|---|---|---|
| `timestamp` | RFC 3339 nano string | When the shipper saw the line (close to when Docker emitted it). |
| `project` | string | Same as the routing-key segment. |
| `service` | string | Same as the routing-key segment. |
| `container_name` | string | Docker container name (e.g. `my-looky-app-1`). Useful to differentiate replicas. |
| `container_id` | string | Short (12-char) Docker container ID. |
| `image` | string | Image reference the container was started from. |
| `stream` | string | `stdout` or `stderr`. |
| `message` | string | The raw line as Docker emitted it. **No parsing happens** — if a container emits JSON, this is a string containing JSON; the consumer parses it (or doesn't). |

## Subscribing

The shipper only publishes; it does **not** create queues. Consumers
declare their own queues and bind them to `containers-logs` with
whatever routing-key pattern they need. A debug queue
`containers-logs.debug` bound to `#` is the only one that exists by
default.

## The shipper

`karotten` is a small Go daemon that runs on the host (outside this
stack) and publishes container stdio into `containers-logs` per the
contract above. Source, build, deploy, env vars, and resilience design
all live in its own repo: <https://github.com/paranoid-software/karotten>.
