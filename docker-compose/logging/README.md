# Host-level log shipper — install + setup

This directory holds the Vector config and the install steps for
shipping every container's stdout to RabbitMQ. Architecture + contract
are documented internally in the looky workspace
(`SPEC_LOG_SHIPPER.md`); this README is the **operator-facing recipe**
for deploying the shipper alongside the looky-distro stack.

Two host environments are supported, with the same `vector.yaml`:

- **Mac dev** — Vector via Homebrew, talking to the Docker Desktop socket.
- **Ubuntu ARM (OCI prod)** — Vector via apt, talking to the host Docker socket.

Pick your section below.

---

## 0. RabbitMQ one-time setup (both environments)

Done once per broker. Adjust `BROKER` to match the container name of
your rabbit instance:

```bash
BROKER=rabbitmq-4-testing   # dev local; change for prod

# Write-only user, scoped to logs-upstream.*
docker exec $BROKER rabbitmqctl add_user looky_logs <PASSWORD>
docker exec $BROKER rabbitmqctl set_permissions \
  -p looky looky_logs "" "logs-upstream\..*" ""

# Topic exchange
docker exec $BROKER rabbitmqadmin declare exchange \
  --vhost=looky name=logs-upstream type=topic durable=true \
  -u guest -p guest
```

Save `<PASSWORD>` — you'll set it as `LOOKY_LOGS_PASSWORD` in §1 or §2.

Verify:
```bash
docker exec $BROKER rabbitmqadmin list exchanges --vhost=looky
# logs-upstream | topic | durable=true
```

---

## 1. Mac dev — install steps

```bash
# 1.1 Install
brew install vectordotdev/brew/vector
vector --version

# 1.2 Place the config
mkdir -p ~/.config/vector
cp /path/to/looky-distro/docker-compose/logging/vector.yaml ~/.config/vector/vector.yaml

# 1.3 Set env vars (append to ~/.zshrc or ~/.bashrc)
cat >> ~/.zshrc <<'EOF'
export DOCKER_HOST="unix://$HOME/.docker/run/docker.sock"
export RABBITMQ_HOST="localhost"
export LOOKY_LOGS_PASSWORD="<paste from §0>"
EOF
source ~/.zshrc

# 1.4 Test in foreground (Ctrl-C when satisfied)
vector --config ~/.config/vector/vector.yaml
# Expected: "Vector has started" + "started healthcheck"; no AMQP errors.

# 1.5 Run as a service
brew services start vector
brew services list | grep vector   # status: started
```

Vector's own log:
```bash
tail -f /opt/homebrew/var/log/vector.log
```

Reload after editing `~/.config/vector/vector.yaml`:
```bash
brew services restart vector
```

---

## 2. Ubuntu ARM (OCI prod) — install steps

OCI Ampere instances are aarch64; Vector publishes arm64 packages
through its apt repo, so the standard install works.

```bash
# 2.1 Add the apt repo + install
curl -1sLf 'https://repositories.timber.io/public/vector/cfg/setup/bash.deb.sh' \
  | sudo -E bash
sudo apt update
sudo apt install vector
vector --version
# Expected: vector 0.50.x or newer, target aarch64-unknown-linux-gnu

# 2.2 Place the config
sudo install -o root -g root -m 0644 \
  /path/to/looky-distro/docker-compose/logging/vector.yaml /etc/vector/vector.yaml

# 2.3 Set env vars in /etc/default/vector (the apt package's standard
# place for systemd unit overrides)
sudo tee /etc/default/vector > /dev/null <<'EOF'
DOCKER_HOST=unix:///var/run/docker.sock
RABBITMQ_HOST=<host_or_ip_of_broker>
LOOKY_LOGS_PASSWORD=<paste from §0>
EOF
sudo chmod 0640 /etc/default/vector
sudo chown root:vector /etc/default/vector   # only vector + root read it

# 2.4 Vector's apt package runs as user `vector`. Add it to the
# docker group so it can read /var/run/docker.sock.
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

---

## 3. Verification (either environment)

Bind a tap queue that receives everything, generate some traffic,
read what landed:

```bash
BROKER=rabbitmq-4-testing
RMQ="docker exec $BROKER rabbitmqadmin --vhost=looky -u guest -p guest"

# Tap queue
$RMQ declare queue name=test-tap durable=false
$RMQ declare binding source=logs-upstream destination=test-tap routing_key="#"

# Generate a log line (4xx is fast)
curl -X POST http://localhost:3002/probe \
  -H 'Content-Type: application/json' \
  -d '{"sourceAlias":"none"}'

# Read what arrived
$RMQ get queue=test-tap count=10
```

Expected: JSON events with `container_name`, `level`, `message`, and
the looky canonical shape for looky containers (extra fields like
`request_id`, `workspace_id`, etc.).

If nothing arrives:

| Symptom | Check |
|---|---|
| Vector not running | `brew services list \| grep vector` (mac) / `systemctl status vector` (linux) |
| AMQP errors in Vector log | password / RABBITMQ_HOST / vhost permissions (re-run §0) |
| Vector running but no enrichment fields | `DOCKER_HOST` value, the unix socket file actually exists at that path |
| Vector running but cannot read socket | `vector` user not in `docker` group (linux), or Docker Desktop not running (mac) |

---

## 4. Subscribing — common patterns

After Vector is shipping, declaring a stream is two `rabbitmqadmin`
commands and zero Vector changes. The `vector.yaml` itself is
immutable from this point.

```bash
RMQ="docker exec $BROKER rabbitmqadmin --vhost=looky -u guest -p guest"

# Pattern: only looky errors
$RMQ declare queue name=looky-errors durable=true
$RMQ declare binding source=logs-upstream destination=looky-errors \
  routing_key="looky-dev-app-1.error"
$RMQ declare binding source=logs-upstream destination=looky-errors \
  routing_key="looky-dev-query-engine-1.error"
$RMQ declare binding source=logs-upstream destination=looky-errors \
  routing_key="looky-dev-export-scheduler-1.error"
$RMQ declare binding source=logs-upstream destination=looky-errors \
  routing_key="looky-dev-export-worker-1.error"

# Pattern: any error from any container
$RMQ declare queue name=any-errors durable=true
$RMQ declare binding source=logs-upstream destination=any-errors \
  routing_key="*.error"

# Pattern: full stream of a single container (e.g. mongo)
$RMQ declare queue name=mongo-stream durable=true
$RMQ declare binding source=logs-upstream destination=mongo-stream \
  routing_key="oom-mongo.#"
```

Same queue can carry multiple bindings — RabbitMQ unions them.

---

## 5. Updating the config

Edit `looky-distro/docker-compose/logging/vector.yaml` in the repo. Then on each host:

- **Mac**: `cp` to `~/.config/vector/vector.yaml` + `brew services restart vector`.
- **Linux**: `sudo cp` to `/etc/vector/vector.yaml` + `sudo systemctl restart vector`.

Vector also supports `--watch-config` for hot reload without restart;
see the upstream docs if you want that.
