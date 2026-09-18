# NVIS-P2P Console Workflow

## Open and secure the console

1. Create the SSH tunnel from the workstation:
   `ssh -N -L 4301:192.168.10.3:4300 appserver`
2. Open `http://localhost:4301`.
3. Sign in as `root`. The initial vendor password is `dragonfly`.
4. If the initial password is still active, open the user menu, select
   **Profile**, open **Change Password**, and rotate it into the approved secret
   store. The Artifact Fleet controls are visible only to the root role.
5. Select **Artifact Fleet** in the left navigation.

Port 4300 is limited to appserver loopback and its management address. It is not
published on a compute-facing address.

## Read the dashboard first

The cards across the top summarize the latest preflight observation:

- **Object store** must be `Healthy` before importing artifacts.
- **Reachable nodes** shows nodes that answered the bounded SSH preflight.
- **Eligible nodes** excludes unreachable nodes, changed host keys, wrong
  architecture/page size, incorrect service identity, and unwritable targets.
- **Persistent peers** shows eligible nodes with a responding dfdaemon socket.
- **Full-catalog ready** counts nodes whose files match every entry in the
  current catalog. The final Verify operation also checks their SHA-256
  receipts. The count is derived from the catalog rather than hard-coded.

The **SU readiness** table provides the same values per service unit. A node
excluded by preflight is not modified by later operations.

## Initial setup or expansion

Run these controls in order. Only one fleet operation can run at a time.

1. Under **Select service units**, select the SUs to manage. Use **All** for
   SU1-SU14.
2. Click **A. Preflight** and confirm. Wait for the blue running banner to
   disappear. Open the operation log and confirm `PREFLIGHT_COMPLETE`.
3. Review Reachable and Eligible. Investigate exclusions instead of bypassing
   SSH host-key failures or changing broad permissions.
4. Click **B. Install peers** and confirm. This installs the small aarch64
   `dfdaemon` and `dfget` binaries plus `nvis-p2p-peer.service` on eligible
   nodes. It does not install a container engine, GPU software, or a monitoring
   stack.
5. Wait for `DEPLOY_COMPLETE` in the operation log. Run **A. Preflight** again
   so Persistent peers reflects the deployment.

## Persist the complete catalog

1. Keep the intended SUs selected.
2. In **Select canonical images**, select every row in the current catalog.
3. Click **C. Persist selected** and confirm.
4. Wait for the operation to finish. The controller imports each catalog image
   sequentially through `dfstore`, writes it to the NVIS S3-compatible object
   store, and requests 28 Dragonfly persistent replicas with a seven-day TTL.
5. Confirm that every row changes from `Not imported` to `Imported`, and check
   that the operation log contains one `PERSIST_READY` line per catalog row.

The stock `nvidia+nemo+26.04.01-arm.sqsh` is a distinct catalog image. It does
not replace the canonical NeMo/NCCL or Llama images.

## Sync the complete catalog with native dfstore

After every catalog image has been persisted, use the service-driven sync wave:

1. Select the intended SUs. The backend intersects this selection with the
   preflight eligible-node allow-list; excluded nodes are never contacted.
2. Click **D. Sync catalog** and confirm.
3. Open the operation log. The controller queues
   `nvis-p2p-catalog-sync.service` with non-blocking `systemctl` calls in at
   most 48 concurrent SSH sessions. The service performs the dfstore work;
   the controller contains no transfer implementation.
4. Watch `SYNC_WAVE_PROGRESS`. Each poll is a new, short SSH command and reads
   only systemd service state and
   `/opt/containers/.nvis-p2p/logs/catalog-sync.log`. It never reads the
   systemd journal.
5. A node is complete only after its log reports
   `NVIS_P2P_SYNC_COMPLETE ... failed=0` after the current wave marker.
6. `SYNC_WAVE_COMPLETE_WITH_DEFERRALS` is an explicit partial outcome. Open
   the reported run directory and inspect `deferred.tsv`; `complete.tsv`
   contains the successful nodes. Unreachable nodes are checked three times,
   service failures twice, and the overall wave is bounded to 24 hours.
7. Click **E. Verify fleet** after the wave completes.

Only one console operation can run at a time, so a sync wave cannot overlap a
deployment, persistence import, verification, or another sync wave.

## Legacy single-image distribution

Distribution deliberately accepts exactly one selected image per operation.

1. Clear four image checkboxes, leaving one image selected.
2. Select the target SUs.
3. Click **Legacy single-image rollout** and confirm.
4. Open the operation log. The rollout controller checks nodes in bounded
   batches and reports `ROLLOUT_PROGRESS`. Existing correct files with a valid
   receipt are not recopied. A mismatched file is preserved in quarantine
   before replacement. Transient failures are retried at most three times.
5. Wait for `ROLLOUT_COMPLETE`, or `ROLLOUT_COMPLETE_WITH_DEFERRALS`. The
   latter is not silent success: its run directory contains `deferred.tsv`
   with every unreachable or retry-exhausted node and the exact reason.
   `ready.tsv` lists every node that had both the expected size and receipt.
6. Click **E. Verify fleet** after the rollout. Open its log and use
   the linked TSV path to distinguish ready, active, missing, bad-size, and
   unverified nodes.
7. Repeat steps 1-6 for each remaining image. Do not select multiple images for a
   distribution run; the console rejects that request to prevent simultaneous
   multi-image hashing and writes across the fleet.

## Repair drift

1. Select only the affected image and affected SUs.
2. Click **Repair selected image**.
3. Wait for completion and then click **E. Verify fleet**.

Repair uses the same preflight allow-list and content digest as normal
distribution. It never overwrites a mismatched file in place.

## What each control changes

| Control | Changes compute nodes | Changes canonical files | Uses GPU | Queries systemd journal |
|---|---:|---:|---:|---:|
| Preflight | No | No | No | No |
| Install peers | Installs/starts NVIS-P2P peer service | No | No | No |
| Persist selected | No | Reads only | No | No |
| Sync catalog | Runs native `dfstore export` for every catalog row and writes verified files under `/opt/containers` | No | No | No |
| Distribute one image | Writes selected file under `/opt/containers` | No | No | No |
| Verify fleet | No | No | No | No |
| Repair selected image | Repairs only the selected file | No | No | No |

## Operational rules

- Do not run backend distribution commands while a console operation is active.
- Treat host-key changes as an identity issue; do not disable host-key checking.
- Investigate nodes excluded because `/opt/containers` is not writable; do not
  make the directory broadly writable.
- Use the operation log as the source of truth. The dashboard refreshes every
  15 seconds.
- Re-run Preflight after peer deployment and after large maintenance events so
  the readiness cards reflect current node state.
