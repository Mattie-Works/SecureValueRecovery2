# Standard Linux / Docker deployment (non-TEE)

This guide runs the SGX-free SecureValueRecovery2 fork on ordinary Linux without
AWS Nitro Enclaves or AMD SEV-SNP. It uses the existing **simulated Nitro**
transport to launch the recovery engine as a normal Linux process.

> **Security warning:** This is a development/test deployment, **not** a
> hardware-backed enclave. Root, a compromised kernel, and privileged container
> operators may inspect or modify process memory and bypass security assumptions
> underlying PIN attempt limits. Do not use this deployment for real recovery
> secrets or claim TEE attestation. Redis is a peer-discovery service, not
> durable backup storage. Single-node Raft cannot tolerate node loss.

## Prerequisites

- Linux x86-64 with Docker Engine, Docker Compose v2, and Docker Buildx
- Git, GNU Make, and adequate free disk/RAM for the toolchain build
- Outbound access for downloading pinned Go, Rust, LLVM, and dependencies
- No SGX, Nitro hardware, or AWS account is required

Use the `codex/remove-intel-sgx` branch (the Docker builder includes the
`wget` fix). From the repository root:

```sh
git checkout codex/remove-intel-sgx
git pull origin codex/remove-intel-sgx
git submodule update --init --recursive
make docker_enclave
make docker_host
test -x host/main && test -x enclave/build/enclave.nsm
```

These commands build `host/main` and `enclave/build/enclave.nsm`. Building
and running the application have **not** been verified end-to-end in this fork.

## Configure the local deployment

Create a deployment directory:

```sh
mkdir -p deploy/standard
cp host/integration/testdata/enclave.config deploy/standard/enclave.config
```

The copied protobuf-text configuration has `group_config.simulated: true`.
Keep it enabled **only for development**. It also sets
`min_voting_replicas: 1`, which is suitable for a local smoke test but not
a resilient cluster.

Create `deploy/standard/host.yaml`:

```yaml
peerAddr: "127.0.0.1:9000"
clientListenAddr: "0.0.0.0:8080"
controlListenAddr: "127.0.0.1:8081"
enclaveId: "enclave"
redis:
  name: "svr2-standard"
  addrs:
    - "127.0.0.1:6379"
raft:
  tickDuration: 1s
  initialJoinDuration: 1m
otlpEnabled: false
```

All omitted fields use defaults in `host/config/config.go`. The Redis
address assumes Linux **host networking**. If using bridge networking,
change the addresses and expose ports accordingly.

Create `deploy/standard/Dockerfile`:

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates libstdc++6 libgcc-s1 curl \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY host/main /app/host
COPY enclave/build/enclave.nsm /app/enclave
COPY deploy/standard/host.yaml /app/host.yaml
COPY deploy/standard/enclave.config /app/enclave.config
EXPOSE 8080 8081 9000
ENTRYPOINT ["/app/host"]
CMD ["-enclave_type=nitro", "-nitro_path=/app/enclave", "-nitro_port=27427", "-hconfig_path=/app/host.yaml", "-econfig_path=/app/enclave.config"]
```

The image assumes the binaries are dynamically compatible with
`debian:bookworm-slim`. Verify with `ldd host/main` and
`ldd enclave/build/enclave.nsm` before deploying. Missing libraries
must be added to the runtime image; the proposed Dockerfile is not
a validated release artifact.

Create `deploy/standard/compose.yaml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    network_mode: host
    command: ["redis-server", "--bind", "127.0.0.1", "--save", "", "--appendonly", "no"]

  svr2:
    build:
      context: ../..
      dockerfile: deploy/standard/Dockerfile
    restart: unless-stopped
    network_mode: host
    depends_on:
      - redis
    environment:
      AUTH_SECRET: ${AUTH_SECRET:?Set AUTH_SECRET in deploy/standard/.env}
    healthcheck:
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:8081/health/ready"]
      interval: 30s
      timeout: 5s
      start_period: 60s
      retries: 3
```

Redis is intentionally ephemeral here; do not confuse its contents with
durable replicated secret state. Both containers share the host network.
Bind client access behind a firewall or reverse proxy, and never expose
the unauthenticated control interface externally.

Create a local environment file, **never commit it**:

```sh
printf 'AUTH_SECRET=%s\n' "$(openssl rand -base64 32)" > deploy/standard/.env
chmod 600 deploy/standard/.env
```

Add `deploy/standard/.env` to your local Git excludes or repository
`.gitignore` before staging changes. Reuse a securely stored secret
across restarts and hosts that validate the same tokens.

## Start and check

Run from the repository root:

```sh
docker compose --env-file deploy/standard/.env -f deploy/standard/compose.yaml up -d --build
docker compose -f deploy/standard/compose.yaml logs -f svr2
curl -i http://127.0.0.1:8081/health/live
curl -i http://127.0.0.1:8081/health/ready
```

`/health/ready` returns HTTP 200 only after the Raft group has joined or
initialized. The client WebSocket endpoint is
`ws://127.0.0.1:8080/v1/enclave`; it requires the application's
authentication scheme and protobuf protocol, not JSON REST.

To stop:

```sh
docker compose -f deploy/standard/compose.yaml down
```

**Do not treat this as persistent secret storage:** shutting down the
single-node recovery engine loses its in-memory Raft state. Recreating
containers may make existing recovery data unavailable.

## Alternative: run directly on Linux

Start a local Redis server on `127.0.0.1:6379`, then:

```sh
export AUTH_SECRET="$(openssl rand -base64 32)"
./host/main \
  -enclave_type=nitro \
  -nitro_path=./enclave/build/enclave.nsm \
  -nitro_port=27427 \
  -hconfig_path=./deploy/standard/host.yaml \
  -econfig_path=./deploy/standard/enclave.config
```

The host starts the simulated enclave as a child process, using TCP
`127.0.0.1:27427`. No Nitro daemon is involved.

## Before a real production deployment

1. Design and implement a supported **non-TEE security model**; the
   simulated backend must not silently masquerade as attested hardware.
2. Use a multi-node Raft deployment, test recovery after process and
   host failures, and establish backup/disaster-recovery procedures.
3. Secure client authentication, TLS, Redis access, and Raft peer networking.
4. Validate binary runtime dependencies and automated build/integration tests.
5. If TEE guarantees are required, use an attested confidential VM
   (e.g. SEV-SNP) and update client trust/attestation policy accordingly.

See also [SGX removal notes](SGX_REMOVAL.md) and
[`host/integration/testdata/enclave.config`](../host/integration/testdata/enclave.config).


## Troubleshooting: Docker networking, Redis Cluster and startup

The initial host-network/standalone-Redis example above is insufficient for
SVR2's Redis peer discovery: `host/peer/peerdb/peerdb.go` uses
`redis.NewClusterClient`. Prefer a **private Docker bridge network** and
Redis Cluster, even for local testing. Do not advertise the host LAN address
(`192.168.108.134`) when both services communicate inside Compose.

### Recommended bridge-network example

Use these *replacement* settings in `deploy/standard/host.yaml`:

```yaml
peerAddr: "svr2:9000"
clientListenAddr: "0.0.0.0:8080"
controlListenAddr: "0.0.0.0:8081"
enclaveId: "enclave"
redis:
  name: "svr2-standard"
  addrs: ["redis:6379"]
raft:
  tickDuration: 1s
  initialJoinDuration: 1m
otlpEnabled: false
```

Use this Compose example instead of the host-network example above:

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    hostname: redis
    command:
      - redis-server
      - --bind
      - "0.0.0.0"
      - --port
      - "6379"
      - --cluster-enabled
      - "yes"
      - --cluster-config-file
      - nodes.conf
      - --cluster-node-timeout
      - "5000"
      - --cluster-announce-hostname
      - redis
      - --cluster-preferred-endpoint-type
      - hostname
      - --appendonly
      - "yes"
      - --protected-mode
      - "no"
    volumes:
      - redis-data:/data
    networks: [svr2-net]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  svr2:
    build:
      context: ../..
      dockerfile: deploy/standard/Dockerfile
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
    environment:
      AUTH_SECRET: ${AUTH_SECRET:?Set AUTH_SECRET in deploy/standard/.env}
    ports:
      - "127.0.0.1:8080:8080"
    networks: [svr2-net]
    healthcheck:
      test: ["CMD", "curl", "--fail", "--silent", "http://127.0.0.1:8081/health/ready"]
      interval: 30s
      timeout: 5s
      start_period: 60s
      retries: 3

networks:
  svr2-net:
    driver: bridge

volumes:
  redis-data:
```

The Redis service name is a Docker DNS hostname. Redis Cluster also advertises
topology endpoints to clients, so the advertised endpoint must resolve and be
reachable **from the SVR2 container**. Hostname announcements require compatible
Redis and Go client versions; verify `CLUSTER SLOTS` and client behavior.
`protected-mode no` is safe only on a restricted, unexposed development
network. No Redis host port is published.

### Error: `Address already in use` on Nitro simulation port 27427

```text
socketmain/socketmain.cc ... bind ... errno(98): Address already in use
Nitro simulated command finished: signal: aborted
```

The simulated enclave listens on TCP 27427. With `network_mode: host`,
another process or container can own the same port. Inspect:

```sh
sudo ss -ltnp '( sport = :27427 )'
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Prefer bridge networking (the enclave and host still share the *same* SVR2
container network namespace). Alternatively change `-nitro_port` to an
unused port such as 27428. If the conflict persists under bridge networking,
check for duplicate child processes inside the SVR2 container.

### Error: `ERR This instance has cluster support disabled`

This means the Redis server is running in standalone mode while SVR2 uses
`redis.NewClusterClient`. Enable `--cluster-enabled yes`, persist
`nodes.conf` in a Redis volume, and initialize hash slots on a fresh
single-node development cluster:

```sh
docker compose -f deploy/standard/compose.yaml up -d redis
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster addslotsrange 0 16383
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster info
```

Only assign slots if they are not already assigned. This one-node approach
is for debugging and may not reach a healthy cluster state on all Redis
versions/configurations.

### Error: `dial tcp :6379: connect: connection refused`

The missing host in `:6379` strongly suggests a malformed or missing
advertised Redis Cluster endpoint, *not* necessarily a Docker DNS failure.
Inspect the topology:

```sh
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster nodes
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster slots
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster shards
docker compose -f deploy/standard/compose.yaml logs --tail=100 redis
```

Check that `redis:6379` resolves inside the SVR2 network and that the
Redis-advertised endpoint is nonempty. Redis may return an empty or `?`
endpoint if it cannot determine an address. Setting
`--cluster-announce-hostname redis` and
`--cluster-preferred-endpoint-type hostname` may resolve this, provided
the client supports hostname endpoints. If not, use a **stable private
Docker IP** announced by Redis and keep the network private; do not
publish unauthenticated Redis on the host LAN.

### Error: `cluster_state:fail` with all 16384 slots assigned

Example:

```text
cluster_state:fail
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:1
cluster_size:1
```

All slots assigned does **not** prove that Redis considers the cluster
healthy. Inspect `CLUSTER NODES`, `CLUSTER INFO`, and server logs to
identify the reason. A single master is not a recommended Redis Cluster
topology; prefer at least **three masters** for functional cluster testing.
Three masters without replicas are *not* highly available. Production
typically needs replicas and appropriate failure-domain separation.

Do **not** treat `cluster-require-full-coverage no` or
`cluster-allow-reads-when-down yes` as fixes for cluster health: the latter
does not make failed-cluster writes succeed.

For a fresh three-master setup, after provisioning `redis-1`,
`redis-2`, and `redis-3` on the same private network, initialize from
a container that can resolve all three service names:

```sh
redis-cli --cluster create redis-1:6379 redis-2:6379 redis-3:6379 --cluster-replicas 0
```

Then set `redis.addrs` to all three hostnames and confirm
`cluster_state:ok`. Each Redis master must announce an address reachable
from the others and SVR2; the cluster bus ports must be reachable between
Redis nodes. This is an architecture example, not a prebuilt Compose stack.

### Restart and health verification

```sh
docker compose --env-file deploy/standard/.env -f deploy/standard/compose.yaml up -d --force-recreate redis svr2
docker compose -f deploy/standard/compose.yaml logs -f svr2
docker compose -f deploy/standard/compose.yaml exec redis redis-cli cluster info
```

If control port 8081 is not published, test readiness **inside** SVR2:

```sh
docker compose -f deploy/standard/compose.yaml exec svr2 \
  curl -i http://127.0.0.1:8081/health/ready
```

The `redis-cli ping` health check tests basic Redis responsiveness, **not**
Redis Cluster readiness. An SVR2 process can be alive while Raft readiness
is still pending. If you change cluster topology, keep existing persistent
Redis data unless you have explicitly decided to discard development state.
