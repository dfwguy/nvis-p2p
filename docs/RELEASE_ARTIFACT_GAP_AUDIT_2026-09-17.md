# NVIS-P2P release and deployment artifact gap audit

Date: 2026-09-17  
Scope: read-only review of `/home/hlakshminara/NVIS-P2P`, Dragonfly `v2.5.1`
source, Dragonfly client `v1.4.2` source, downloaded releases, local scripts,
configuration, inventory, and captured state. No service, Slurm, node, or
running-process changes were made.

## Executive verdict

The directory is a successful SU1 prototype, not yet a releasable or
repeatable deployment package. The captured evidence says all 288 SU1 nodes
were reachable and all 288 peers were healthy. The binaries identify as the
expected upstream releases, shell syntax passes, and all six YAML files parse.
Those are useful foundations.

It is **not ready** for the requested Git-ready package, deployable tarball,
user/deployment guide, or one-shot `SKILL.md`. The blockers are:

1. The root is not a Git repository and has no root README, license policy,
   `.gitignore`, release manifest, tests, changelog, or version.
2. `configs/manager.yaml` contains a literal database password and a literal
   JWT signing key. These must not enter any Git history or release artifact.
3. There is no reproducible control-plane deployment. PostgreSQL, Redis,
   manager, scheduler, manager database bootstrap, scheduler-cluster bootstrap,
   seed-cluster bootstrap, and seed-peer lifecycle are absent from the scripts.
4. Manager, scheduler, and seed are currently manual long-running processes;
   compute peers are `nohup` processes. There are no service definitions,
   restart policy, graceful stop, upgrade, rollback, or uninstall paths.
5. The custom `client-arm64-64k` binaries have no complete build recipe or
   provenance record. The Dockerfile alone does not record source commit,
   build command, builder digest, output collection, or reproducibility test.
6. Fleet rollout is not safely orchestrated. Wave files exist, but nothing
   enforces canary success, wave completion, error budget, load limits, pause,
   resume, or rollback before the next wave.
7. The package is tied to one user, one appserver address, SU1, one image name,
   and one origin path. It cannot be installed in another site without editing
   source files.

Do not create the final bundle or tar until the P0 items below are closed and a
secret scan proves the credentials were never committed.

## Evidence and compatibility baseline

### Pinned upstream sources

| Component | Tag | Commit | Local evidence |
|---|---:|---|---|
| Dragonfly core | `v2.5.1` | `5523218a3710b2d5bc8fbdb0b1473d3a4d64916d` | detached clean source checkout; manager and scheduler report `v2.5.1`, commit `5523218a` |
| Dragonfly Rust client | `v1.4.2` | `c2f7edbfd4df573192beca60ca18790a730740e4` | detached clean source checkout; amd64 `dfdaemon` reports `1.4.2`, commit `c2f7edbfd` |

The core tar hash, `361494a6782db9240c89352f6b2f115c012ebe96cb0039c1527610ade0514804`,
matches the included upstream checksum list. The two client release tar hashes
match the digests in the captured GitHub release metadata:

- aarch64 GNU: `90214f04cdfcace2fe2e30edf276ccde68eeb3021f2d047fbea768c56a7ece41`
- x86-64 GNU: `59fb078256c0c34702f0d6fd57324d36be14504461d1b23a238966cb56c7054e`

The deployed custom binaries are ARM64, dynamically linked, and have 64 KiB
ELF load alignment. Their hashes are already hard-coded into the deploy
script, but no signed or reproducible provenance document explains how they
were derived. Notably, the upstream aarch64 release binary also has 64 KiB
load alignment; the guide must explain the functional reason for selecting the
custom build and show an actual 64 KiB host runtime test, not merely an ELF
alignment check.

### Prototype evidence

- `state/SU1-preflight.tsv`: 288 reachable nodes.
- `state/SU1-peer-audit.tsv`: 288 healthy peers.
- Six wave inventories contain 48 nodes each.
- All scripts pass `bash -n`.
- All YAML files parse as YAML.
- Manager, scheduler, and seed processes are presently running, and local
  PostgreSQL/Redis containers exist, but their creation and lifecycle are not
  represented in this directory. Runtime existence is not deployment evidence.

The state files prove a point-in-time SU1 result. They must be retained as
redacted release evidence, not treated as an installation mechanism.

## P0 release blockers

### 1. Remove secrets before Git initialization

`configs/manager.yaml` has a static JWT signing key and a static PostgreSQL
password. Replace literals with a generated, mode-`0600` deployment environment
or Docker/Podman secret files. Generate unique secrets at install time. Never
print them, put them on a command line, store them in state reports, or provide
defaults that work in production.

Add a release secret scanner covering the worktree, staged content, Git object
history, tar members, documentation, logs, and configuration. It must reject
known prototype values, password/key assignments, private keys, bearer tokens,
`sshpass -p`, and credential-bearing URLs.

Because the root currently has no `.git`, this can still be fixed without
polluting history. Initialize Git only from a sanitized staging tree.

### 2. Build a complete, idempotent control-plane deployment

The release needs a checked-in Compose or systemd design for:

- PostgreSQL 16 with persistent storage, healthcheck, protected credentials,
  backup/restore, schema migration, and loopback/internal-only exposure;
- Redis 7 with persistence policy, authentication or isolated internal
  networking, healthcheck, and bounded memory policy;
- manager `v2.5.1` with durable logs/state and readiness check;
- scheduler `v2.5.1` with durable data and readiness/registration check;
- amd64 seed `dfdaemon v1.4.2` with durable cache and readiness check;
- explicit CPU, memory, open-file, log-rotation, stop-grace, and restart limits;
- an isolated network and a documented, minimal list of published ports.

Fresh manager databases do not automatically contain the site topology implied
by `schedulerClusterID: 1`. Add an idempotent bootstrap step that creates or
reconciles the scheduler cluster, seed-peer cluster, scheduler, and seed-peer
association through supported manager APIs. It must discover IDs rather than
assuming database row `1` and must be safe to run twice.

`configs/scheduler-dynconfig.yaml` is not used when the scheduler has a manager
address, per the upstream `scheduler --help`. `configs/seed-dynconfig.yaml` is
also not referenced by the seed config or launch command. Either wire these
files into a documented supported local-config mode or remove them from the
operator path and bootstrap equivalent manager state.

### 3. Replace manual and `nohup` process lifecycle

Create service units or rootless service supervision for control-plane and peer
processes. Each unit needs:

- deterministic user/group and directory ownership;
- `ExecStart`, graceful `ExecStop`, bounded restart/backoff, and startup order;
- `LimitNOFILE`, CPU/IO priority, CPU affinity policy, and resource limits;
- PID ownership from the supervisor rather than ad-hoc PID files;
- log rotation and disk limits;
- reboot persistence and an explicit opt-out;
- health/readiness checks that verify scheduler registration, not just a PID;
- versioned config and binary paths supporting atomic upgrade and rollback.

Hard-coded `taskset -c 0-15` is unsafe as a universal default. Preflight must
verify the allowed cpuset and reserve a configurable CPU set that does not
conflict with workload policy. If no reservation exists, refuse deployment.

### 4. Implement safe canary/wave orchestration

The one-shot installer must implement a state machine, not invoke the existing
scripts back-to-back:

1. Validate immutable inventory, target uniqueness, DNS/IP mapping, SSH host
   keys, architecture, 64 KiB page size, libc/loader, free inodes/bytes,
   allowed cpuset, required ports, and origin isolation.
2. Deploy one canary per relevant rack or failure domain.
3. Run a small known-object transfer and verify full SHA-256, parent/seed path,
   absence of compute-node origin access, scheduler health, CPU, network, and
   disk thresholds.
4. Advance one bounded wave at a time only if the previous wave reaches a
   configurable success threshold and the seed/control-plane remain below
   limits.
5. Persist a signed/hashed checkpoint after each wave, allowing resume without
   redeploying completed nodes.
6. Stop automatically on error-budget, seed saturation, scheduler errors,
   origin requests from compute nodes, host load, or disk pressure.
7. Roll back only the release-owned paths and services on failed nodes; never
   delete `/opt/containers`, existing images, unrelated services, or caches
   without an explicit separate purge operation.

The current `launch-wave.sh` does not wait for completion or gate the next wave.
The current fleet scripts accept stale TSV preflight data. Bind preflight to an
inventory hash, configuration hash, binary manifest, timestamp/max age, and
deployment ID.

### 5. Correct integrity checks

`download-llama.sh`, `launch-node.sh`, and `audit-node.sh` accept an existing
target based only on its byte size. A corrupt same-size file is therefore
reported complete. Verify the expected SHA-256 before reuse and after final
placement. Use a same-filesystem temporary path, `fsync` as appropriate, and an
atomic rename. Quarantine invalid files with bounded retention.

Move binary hashes and object metadata out of scripts into a versioned manifest
consumed by preflight, deploy, audit, and download. The manifest should include:

- component name/version/upstream URL/upstream commit;
- file path, size, SHA-256, architecture, libc, ELF interpreter, and page/build
  constraints;
- target object URL, size, SHA-256, application, and tag;
- builder image by digest, `Cargo.lock` hash, source commit, build command,
  `SOURCE_DATE_EPOCH`, and output hashes for custom artifacts.

### 6. Establish SSH and privilege policy

The scripts use `BatchMode` but do not pin a known-hosts file, require strict
host-key checking explicitly, select an SSH user, or validate that inventory
names cannot become SSH options. Add:

- `--`/input validation for every hostname and path;
- `StrictHostKeyChecking=yes`, a regular non-symlink mode-`0600` known-hosts
  file, `UserKnownHostsFile=...`, and fingerprint preflight;
- an explicit non-root deploy account and the minimum documented privilege
  needed for release-owned directories/services;
- no blanket sudo and no password-bearing automation;
- per-node command timeouts, concurrency ceilings, connection rate limits, and
  a kill switch.

The package must define who creates `/opt/containers/.nvis-p2p`, its ownership,
and its mode. Do not rely on whatever permissions happen to exist.

## P1 reliability and operations gaps

### Script correctness and observability

- `preflight-node.sh` emits six fields for an unreachable node under a
  seven-column header. Make schemas machine-validated and emit structured JSON
  plus a human summary.
- `audit-node.sh` verifies only process existence, two hashes, RSS/threads, and
  target size. Also check config hash, service state, socket/ports, scheduler
  registration, peer role, recent errors, disk/cache pressure, target SHA-256,
  download state, and deployed release ID.
- Fleet scripts write directly to final report paths. Write a temporary report,
  validate row count/schema/uniqueness, then atomically publish it.
- Add flock/single-flight protection so two agents cannot deploy or launch the
  same inventory concurrently.
- Add bounded retry with jitter and distinguish unreachable, incompatible,
  deployment-failed, unhealthy, downloading, verified, and rolled-back states.
- Do not discard SSH stderr; retain a sanitized per-node diagnostic and expose
  the actual failure category.
- Add stop, status, resume, upgrade, rollback, and uninstall commands.
- Make rack/location parsing explicit. Existing captured canary location values
  and `${node%-*}` output are inconsistent.
- Parameterize the object; `download-llama.sh` is actually tied to one NeMo
  SquashFS image despite its name.

### Capacity and failure-domain controls

Document and test:

- expected seed NIC throughput, disk read/write throughput, cache footprint,
  peer fan-out, scheduler capacity, and 288-node/wave completion time;
- why peer download/upload limits and parent limits are safe;
- origin connection guarantees and how they are measured;
- behavior when the seed, manager, scheduler, Redis, PostgreSQL, or a parent
  fails mid-transfer;
- restart/resume semantics for partial downloads;
- disk high/low thresholds and coexistence with container image capacity;
- expansion from SU1 to multiple SUs without a single appserver/seed failure
  domain.

The seed is currently a single point of failure and a potential appserver/NFS
contention source. The production design should define at least capacity-tested
HA or a documented degraded-mode/runbook and recovery-time objective.

### Security and transport

Manager, scheduler, peer upload, and metrics endpoints are plaintext in the
reviewed configs. Upstream templates expose TLS/mTLS fields. Either implement
mTLS with a documented CA/certificate rotation workflow or document a reviewed
network-isolation decision and enforce it at the host/network boundary. Bind
administrative APIs to loopback/internal networks and expose only required data
plane ports.

Add dependency/SBOM and vulnerability reports for Go binaries, Rust binaries,
PostgreSQL, Redis, and any container images. Pin container images by digest.

## Required artifact layout

Use a sanitized staging tree as the single source for all deliverables:

```text
NVIS-P2P/
  README.md
  VERSION
  CHANGELOG.md
  LICENSES/
  NOTICE
  SECURITY.md
  .gitignore
  .gitattributes
  configs/examples/
  deploy/{compose,systemd}/
  docs/{architecture,user-guide,deployment,operations,rollback,troubleshooting}/
  inventory/examples/
  manifests/release.lock.json
  scripts/{install,preflight,deploy,verify,status,upgrade,rollback,uninstall}/
  skills/nvis-p2p-deploy/SKILL.md
  skills/nvis-p2p-deploy/{scripts,references}/
  tests/{unit,integration,fixtures}/
```

Keep site inventory/configuration in a separate protected overlay where
possible. Never include live `state/`, `profiles/`, logs, PID files, generated
peer configs, database volumes, cache, or secrets. Exclude nested upstream
`.git` directories. Preserve upstream license/notice files for every distributed
binary.

### Git-ready package

The current `dist/core/manager` is about 120 MB and exceeds GitHub's ordinary
100 MB object limit. Choose and document one of these models:

1. preferred: source/config/scripts/docs in Git; signed binary payload in the
   release tar with a lock manifest;
2. internal Git LFS with server support, a checked-in `.gitattributes`, and an
   offline LFS export test;
3. an internal artifact repository with immutable digest-addressed objects.

Do not silently commit 414 MB of binaries to ordinary Git. A fresh offline
clone/bundle must be tested, `git fsck` must pass, history must contain no
secrets, and `git status` must be clean.

### Deployable tarball

The tar must be deterministic (`--sort=name`, fixed owner/group/mtime), contain
one top-level directory, prohibit absolute paths, `..`, devices, FIFOs, hardlink
escapes, and unsafe symlinks, and include `SHA256SUMS` plus a release manifest.
Test safe extraction as an unprivileged user and install from an offline copy.

The deployable tar may include verified binaries even when the Git package does
not. It should not contain the 13 MB nested source clones, duplicate release
tarballs, prototype runtime state, or build caches unless explicitly offered as
a separate source-compliance bundle.

### User and deployment guide

The guide must cover architecture/data flow, ports, roles, object publishing,
canary/wave progress, status interpretation, pause/resume, integrity validation,
capacity controls, routine operation, upgrade, rollback, disaster recovery,
credential/certificate rotation, and troubleshooting. Include screenshots or
captured terminal examples for a clean install, canary success, wave progress,
health/status, an intentional failure, resume, and rollback. Screenshots must
be freshly generated and redact credentials, tokens, internal data not needed
for operation, and unrelated services.

### One-shot `SKILL.md`

`SKILL.md` should be a thin, deterministic orchestrator over checked-in tested
scripts. It must:

- declare exact prerequisites and a noninteractive sealed-input contract;
- verify artifact hash/signature and safe extraction before execution;
- run a no-change preflight and produce a machine-readable plan;
- stop as `BLOCKED` when credentials, pinned host keys, inventory, capacity
  approval, or required privileges are missing—never prompt or guess;
- deploy the control plane, bootstrap manager state, canary peers, verify a
  small object, then advance checkpointed waves;
- enforce single-flight, timeouts, concurrency ceilings, circuit breakers, and
  release-owned path boundaries;
- prove compute nodes cannot contact the origin directly;
- return a final manifest/report with deployed versions, hashes, node outcomes,
  cutover time, and rollback command;
- be rerunnable and resume from checkpoints without duplicating work.

The skill must not contain credentials, use blanket sudo, disable SSH host-key
verification, modify Slurm, alter unrelated node services, or purge existing
container images.

## Minimum release acceptance suite

Before producing the four final artifacts, require all of the following:

1. Shellcheck and unit tests for parsing, manifests, inventory validation,
   templating, checkpoints, and state transitions.
2. Config schema/startup validation against the exact manager, scheduler, and
   client binaries.
3. Container configuration validation and health tests from empty volumes.
4. Secret scan of files, Git objects, bundle, tar members, and rendered docs.
5. Binary checksum, architecture, loader, version, upstream provenance, and
   SBOM verification.
6. Hermetic mocked SSH tests proving strict host-key enforcement, no prompt,
   bounded parallelism, canary gating, stop-on-threshold, resume, and rollback.
7. A disposable integration test: empty control plane to verified small P2P
   transfer, including manager/bootstrap and service restart.
8. Same-size corrupt target test proving it is rejected by SHA-256.
9. Seed/scheduler/manager failure-injection tests and partial-transfer resume.
10. Safe-extraction test, deterministic rebuild comparison, bundle verification,
    fresh clone, `git fsck`, and clean worktree.
11. Canary on a specifically approved non-production node, followed by a
    separately approved bounded wave. This acceptance item is not authorized by
    this audit and must not be inferred from artifact creation.

## Recommended implementation order

1. Freeze the prototype and copy only intended source files into a new
   sanitized staging directory.
2. Externalize/rotate prototype secrets and add secret/history scanning.
3. Create release manifest, config templating, reproducible binary build/fetch,
   and license/SBOM materials.
4. Implement control-plane deployment/bootstrap and supervised peer lifecycle.
5. Implement integrity-correct download, structured status, checkpointed
   canary/wave deployment, rollback, and tests.
6. Write and exercise the user/deployment guide and touchless skill against a
   disposable environment.
7. Initialize a one-commit sanitized Git repository, create the deterministic
   tar and verified bundle, then generate hashes from the final immutable files.

Until steps 1–5 pass, the existing directory should remain labeled a prototype
and must not be presented as a touchless production installer.
