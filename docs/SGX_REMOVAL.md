# Running without Intel SGX

This fork removes the Intel SGX runtime, Open Enclave SDK, SGX binary release
artifacts, SGX host bindings, and SGX-specific C++ sources. The core recovery
engine, Raft replication, and protobuf application protocol remain in place.

## Supported paths

| Environment | Host option | Isolation / use |
| --- | --- | --- |
| AWS Nitro Enclaves | `-enclave_type=nitro` (default) | Hardware enclave; requires a Nitro-enabled EC2 host and enclave image |
| AMD SEV-SNP | `-enclave_type=sev` | VM-based TEE; requires external trusted-VM launch and attestation |
| Local development | `-enclave_type=nitro`, simulated group config, `-nitro_path=...` | Separate TCP process, **no hardware isolation** |

## Build

On a machine with Docker and Docker Buildx:

```bash
make dockerbase
make docker_enclave
make docker_host
make enclave_release
```

The regular host binary is `host/main`, the simulated Nitro application is
`enclave/build/enclave.nsm`, and the Nitro host runtime image is
`svr2_nsmhost:latest`. `make enclave_release` also assembles Nitro EIF artifacts.
The Nitro runtime image is **not** a drop-in Docker Compose substitute for the
Nitro hardware daemon: it invokes `nitro-cli`, which needs the appropriate EC2
machine and host configuration.

To run locally, compile the project and configure the existing `host/main`
with its normal `-hconfig_path`, `-econfig_path`, and `AUTH_SECRET` settings,
plus `-enclave_type=nitro -nitro_path=../enclave/build/enclave.nsm`. Set
`group_config.simulated: true` in your enclave config. The host starts the
simulated enclave on a local TCP port. This is for testing only.

## Compatibility and security

- **Existing SGX-attesting clients are incompatible.** Update and audit the
  client-side trusted measurements and attestation verification for Nitro or
  SEV-SNP before routing real client secrets to the new service. Changing the
  host flag alone does not migrate the client's trust model.
- Simulated mode provides no trusted execution boundary. Never use it with
  production recovery secrets or present it as equivalent to a TEE.
- Before migrating a live Raft group, validate mixed-version compatibility,
  backup/restore behavior, attestation, and disaster recovery in staging.
  Do not assume that an SGX-backed replica can join the new hardware group.
- Historical protobuf enum numbers for SGX errors are preserved for wire
  compatibility, but the SGX-specific code paths producing those errors are gone.

## Verification checklist

1. `make docker_enclave_test` and `make docker_enclave_valgrind` (C++ tests).
2. `make docker_host` (Go compile, unit and integration tests against simulated Nitro).
3. `make enclave_release` (Nitro runtime image and EIF generation).
4. Test a real Nitro enclave's attestation and client handshake in staging.
5. Confirm no deployment manifests expect `svr2_sgxrun` or `-sgx_path`.

These checks require a Docker-capable build host and suitable hardware for the
production attestation test. Do not mark the migration production-ready until
they succeed.
