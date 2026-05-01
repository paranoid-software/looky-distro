# minion — host-level container log shipper

This directory holds the Vector config and operator recipe for shipping
every Docker container's stdout from this server (`minion`) into the
RabbitMQ topic exchange **`containers-logs`** on vhost **`minion`** of
the in-stack broker (`the-rabbit`).

The shipper itself runs on the host as an apt-installed Vector daemon.
It reads `/var/run/docker.sock`, normalizes each line to JSON, and
publishes one AMQP message per line. Subscribers declare queues against
the exchange with whatever routing-key pattern they need; everything
else is dropped at the broker.

## Routing-key contract

```
<project>.<container_name>.<stream>
```

Always three segments, all derived from Docker-level metadata. The
shipper never looks inside the message body.

- **project** — `com.docker.compose.project` label of the container.
  For containers started outside of compose this falls back to
  `standalone`. Any literal `.` becomes `-`.
- **container_name** — Docker container name with `.` replaced by `-`
  so segment count stays predictable.
- **stream** — `stdout` or `stderr`, straight from the Docker daemon.

Whether a given line happens to be JSON or plain text is a
consumer-side concern. The full event Vector publishes does include
other Docker metadata as fields in the message body (image,
container_id, host, timestamp, labels, …) — those are available to
consumers but **not** part of the routing key.

Topic-exchange wildcards reminder: `*` matches exactly one segment,
`#` matches zero or more.

---

## 1. Broker prerequisites (one-time, on the-rabbit)

`the-rabbit` runs as a docker container on this host
(`flou-the-rabbit` is the actual container name). Drive the setup
through `docker exec`:

```bash
BROKER=flou-the-rabbit

# Vhost dedicated to this server's container logs
docker exec $BROKER rabbitmqctl add_vhost minion

# Write-only user, scoped to the containers-logs exchange
docker exec $BROKER rabbitmqctl add_user minion_logs <PASSWORD>
docker exec $BROKER rabbitmqctl set_permissions \
  -p minion minion_logs "" "containers-logs.*" ""

# Topic exchange the shipper publishes into
docker exec $BROKER rabbitmqadmin declare exchange \
  --vhost=minion name=containers-logs type=topic durable=true \
  -u guest -p guest
```

Save `<PASSWORD>` — it goes into `MINION_LOGS_PASSWORD` in §2.3.

Verify:

```bash
docker exec $BROKER rabbitmqadmin list exchanges --vhost=minion
# containers-logs | topic | durable=true
```

---

## 2. Install Vector on minion (Ubuntu ARM)

OCI Ampere is `aarch64`; Vector publishes arm64 packages through its
apt repo, so the standard install works.

```bash
# 2.1 apt repo + install
curl -1sLf 'https://repositories.timber.io/public/vector/cfg/setup/bash.deb.sh' \
  | sudo -E bash
sudo apt update
sudo apt install vector
vector --version
# Expected: vector 0.50.x or newer, target aarch64-unknown-linux-gnu

# 2.2 Place the config
sudo install -o root -g root -m 0644 \
  /home/ubuntu/code/looky-distro/docker-compose/logging/vector.yaml \
  /etc/vector/vector.yaml

# 2.3 Env vars (the apt unit reads /etc/default/vector)
sudo tee /etc/default/vector > /dev/null <<'EOF'
DOCKER_HOST=unix:///var/run/docker.sock
RABBITMQ_HOST=localhost
MINION_LOGS_PASSWORD=<paste from §1>
EOF
sudo chmod 0640 /etc/default/vector
sudo chown root:vector /etc/default/vector

# 2.4 The vector user needs to read /var/run/docker.sock
sudo usermod -aG docker vector

# 2.5 Enable + start
sudo systemctl enable --now vector
sudo systemctl status vector
```

Vector's own log:

```bash
sudo journalctl -u vector -f
```

Reload after editing `/etc/vector/vector.yaml`:

```bash
sudo systemctl restart vector
```

`RABBITMQ_HOST=localhost` works because `flou-the-rabbit` publishes
`5672` on the host (`0.0.0.0:5672->5672/tcp`). If that mapping ever
changes, point `RABBITMQ_HOST` at whatever address the host can reach.

---

## 3. Verify

The shipper only publishes; it does not create queues. Verify by
checking that Vector is healthy and that the broker is recording
inbound publishes on the exchange.

```bash
# Daemon is up, no AMQP errors in its log
systemctl status vector
sudo journalctl -u vector -n 100 --no-pager | grep -iE 'error|amqp' || echo "no errors"

# Inbound publish rate on the exchange (look for "publish_in" > 0)
docker exec flou-the-rabbit rabbitmqctl list_exchanges \
  --vhost=minion name type message_stats.publish_in_details.rate
```

You can also open the RabbitMQ management UI
(`http://<minion-host>:15672`, log into vhost `minion`, **Exchanges →
containers-logs**) and watch the message-rate sparkline tick up as
containers emit lines.

If nothing is publishing:

| Symptom | Check |
|---|---|
| Vector not running | `systemctl status vector` |
| AMQP errors in vector log | password / `RABBITMQ_HOST` / vhost permissions (re-do §1) |
| Vector running but no events | `DOCKER_HOST` value, the unix socket file actually exists at that path |
| Vector running but socket access denied | `vector` user not in `docker` group — `usermod -aG docker vector`, then `systemctl restart vector` |

---

## 4. Routing-key reference (for whoever subscribes later)

Queues are **not** part of this deployment — Vector only publishes to
the exchange. If/when someone needs a stream of these logs, they
declare their own queue and bind it themselves. The patterns below are
just reference for what those bindings can look like.

| Goal | `routing_key` |
|---|---|
| Everything (firehose) | `#` |
| All stderr from any container, any project | `*.*.stderr` |
| Everything from one compose project | `<project>.#` |
| Only stderr from one project | `<project>.*.stderr` |
| Both streams from one container | `*.<container>.#` |
| Only stderr from one container | `*.<container>.stderr` |
| All standalone (non-compose) containers | `standalone.#` |
| A specific subset of containers/projects | one binding per pattern; RabbitMQ unions them on the same queue |

`<project>` is the `com.docker.compose.project` label
(`docker inspect <container> --format '{{ index .Config.Labels "com.docker.compose.project" }}'`).
`<container>` is the Docker name from
`docker ps --format '{{.Names}}'`. Any literal `.` in either becomes
`-` in the routing key — see the contract at the top.

If you need to filter by severity / logger / request_id / etc., do it
on the consumer side after parsing the message body — those fields are
not in the routing key by design.

---

## 5. Updating the shipper config

Edit `vector.yaml` in this repo, then on minion:

```bash
sudo install -o root -g root -m 0644 \
  /home/ubuntu/code/looky-distro/docker-compose/logging/vector.yaml \
  /etc/vector/vector.yaml
sudo systemctl restart vector
```

Vector also supports `--watch-config` for hot reload without restart;
see the upstream docs if you want that.
