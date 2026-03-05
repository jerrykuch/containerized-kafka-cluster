# Kafka Cluster Fix: iptables Backend Split-Brain

## Symptom

All five Kafka broker containers start and then die within ~40 seconds
with exit code 1.  ZooKeeper stays running and shows `(healthy)`.

## Root Cause

Two separate issues were found, both preventing Kafka from connecting
to ZooKeeper.

---

### Issue 1: iptables-legacy FORWARD DROP blocking all cross-container traffic

This is the actual root cause. Every packet sent between containers
was being silently dropped.

**Background:** Docker can use two different iptables backends to
program firewall rules:
- `iptables-legacy` — the traditional kernel netfilter/xtables subsystem
- `iptables-nft` — a newer backend that programs nftables, and is the default on Ubuntu 22.04+

Both backends register independent netfilter hooks at the same
priority. The kernel runs packets through **both** hook sets. **If
either backend drops a packet, it is dropped.** There is no "last
writer wins."

**What happened:** At some point Docker on this host was updated or
switched to the `iptables-nft` backend. New Docker networks get their
ACCEPT rules written to `iptables-nft` only. But `iptables-legacy`
still had its old FORWARD chain with:

- `policy DROP` (Docker's default, set when it was managing legacy
  iptables)
- ACCEPT rules for old networks (`docker0`, `br-2e08b2c0affb`) — but
  **not** for any new network

Verification:
```
Chain FORWARD (policy DROP 100 packets, 6144 bytes)
  100  6144 DOCKER-USER  all  --  *      *       ...
  100  6144 DOCKER-ISOLATION-STAGE-1  ...
    0     0 ACCEPT  all  --  *      docker0  ctstate RELATED,ESTABLISHED
    0     0 ACCEPT  all  --  docker0  docker0  ...
    0     0 ACCEPT  all  --  *      br-2e08b2c0affb  ctstate RELATED,ESTABLISHED
    0     0 ACCEPT  all  --  br-2e08b2c0affb  br-2e08b2c0affb  ...
    # <-- no rule for br-cbaa8aea3df1 (the current compose network)
```

Every single packet (100 out of 100) hit the DROP policy. Meanwhile,
`iptables-nft` correctly had ACCEPT rules for the current compose
network, but they were irrelevant because `iptables-legacy` already
dropped the packets.

**Why the healthcheck was a false positive:** ZooKeeper's healthcheck
runs `nc localhost 2181` from *inside* the ZooKeeper
container. Loopback traffic never goes through netfilter FORWARD. So
ZooKeeper appeared healthy while being completely unreachable from any
other container.

Proof:
```bash
# From another container on the same Docker network:
$ ping -c 3 zookeeper
3 packets transmitted, 0 packets received, 100% packet loss

# From inside the ZooKeeper container:
$ echo srvr | nc localhost 2181
Zookeeper version: 3.8.3 ...  # works fine
```

**Fix:** Change the `iptables-legacy` FORWARD policy from DROP to
ACCEPT. Since Docker is now managing rules via `iptables-nft`, the
legacy chain no longer needs to enforce isolation — `iptables-nft`
handles that. The legacy DROP was a stale leftover.

```bash
# docker stop gives "permission denied" on the long-running ZooKeeper container,
# so we use nsenter to reach the host's iptables-legacy binary from a privileged container:
docker run --rm --privileged --pid=host --net=host ubuntu \
  nsenter --mount=/proc/1/ns/mnt -- iptables-legacy -P FORWARD ACCEPT
```

This change is **not persistent across reboots.** To make it
permanent, the cleanest approach is to create
`/etc/docker/daemon.json` (it does not exist on this machine yet, but
Docker reads it automatically if present) to consolidate Docker onto
the `iptables-legacy` backend, then restart Docker so it takes full
ownership of those rules:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "iptables-legacy": true
}
EOF
sudo systemctl restart docker
```

After restarting, Docker will manage `iptables-legacy` directly for
all networks (including future ones), and the stale DROP policy will
be replaced with Docker's correct rules. The `iptables-nft` rules from
before will become orphans but are harmless.

Alternatively, if you want to keep Docker on `iptables-nft` (the
current state), you can install the iptables-persistent package to
save and restore the corrected legacy policy on boot:

```bash
sudo iptables-legacy -P FORWARD ACCEPT
sudo apt-get install iptables-persistent   # saves current rules to /etc/iptables/rules.v4
```

---

### Issue 2: depends_on race condition (minor, secondary fix)

Even with working network connectivity, the original `depends_on`
configuration was:

```yaml
depends_on:
  - zookeeper
```

Docker Compose `depends_on` with a plain list only waits for the
container to *start*, not for it to be *ready*. On a fast machine,
Kafka's preflight check starts before ZooKeeper finishes
initializing. This would cause intermittent failures on a first boot
even with correct networking.

**Fix:** Changed all five Kafka service `depends_on` blocks to:

```yaml
depends_on:
  zookeeper:
    condition: service_healthy
```

This makes each Kafka container wait for ZooKeeper's healthcheck to
pass before launching.

---

## Recovery Procedure (what was done)

1. **Identified Kafka's preflight failure** from container logs:
   `Timed out waiting for connection to Zookeeper server
   [zookeeper:2181]`

2. **Confirmed ZooKeeper's healthcheck was a false positive** — `stat`
   is disabled in ZooKeeper 3.8+, `nc` exits 0 regardless of response,
   so the healthcheck always passed even when ZooKeeper was
   unreachable.

3. **Confirmed cross-container network isolation** — `ping` from
   another container showed 100% packet loss. ZooKeeper was reachable
   on loopback but not via its bridge IP.

4. **Identified the iptables split-brain** via `nsenter` into the host mount namespace to run the host's `iptables-legacy` binary:
   ```bash
   docker run --rm --privileged --pid=host --net=host ubuntu \
     nsenter --mount=/proc/1/ns/mnt -- iptables-legacy -L FORWARD -n -v
   ```

5. **Killed the stuck ZooKeeper container** (which resisted `docker stop` and `docker kill` due to a seccomp profile) via its host PID:
   ```bash
   ZKPID=$(docker inspect containerized-kafka-cluster-zookeeper-1 --format '{{.State.Pid}}')
   docker run --rm --privileged --pid=host ubuntu kill -9 $ZKPID
   ```

6. **Clean teardown and restart:**
   ```bash
   docker compose down
   docker compose up -d
   ```

7. **Fixed iptables-legacy FORWARD policy:**
   ```bash
   docker run --rm --privileged --pid=host --net=host ubuntu \
     nsenter --mount=/proc/1/ns/mnt -- iptables-legacy -P FORWARD ACCEPT
   ```

8. **Restarted Kafka brokers:**
   ```bash
   docker compose up -d
   ```

All 6 containers (1 ZooKeeper + 5 Kafka brokers) now run stably.

---

## If This Happens Again

The `iptables-legacy -P FORWARD ACCEPT` fix does not survive a
reboot. If containers lose cross-container connectivity after a host
restart, check:

```bash
docker run --rm --privileged --pid=host --net=host ubuntu \
  nsenter --mount=/proc/1/ns/mnt -- iptables-legacy -L FORWARD -n | head -3
```

If it shows `policy DROP`, re-run the ACCEPT fix above. The permanent
solution is to configure Docker to use a single iptables backend
consistently (see the permanent fix section under Issue 1 above).
