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
