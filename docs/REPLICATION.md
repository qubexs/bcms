# Replication Design (Hospital 1 <-> Hospital 2)

## 1. Database (PostgreSQL assumed – adapt for SQL Server/Oracle)

Default: **streaming physical replication + Patroni/Stolon for HA.**

- Intra-DC: 1 primary + 1 sync standby (zero loss inside DC).
- Inter-DC: 1 async replica in Hospital 2 (cascaded from intra-DC standby to offload primary).
- `archive_mode=on`, WAL archiving to object store shared/mirrored (allows PITR even if streaming lags).
- Logical replication only if per-table filtering needed (e.g., exclude analytics). Avoid for Tier-1 (more lag/fragile).

SQL Server alternative: Always On AG (sync local, async remote). Oracle: Data Guard (SYNC local, ASYNC remote).

Lag SLO: p99 < 60s Tier-1, alert at >5 min, page at >15 min (= RPO breach risk).

## 2. PACS / Imaging (large objects)

- Object storage mirror: MinIO/Ceph bucket replication (versioned) Hospital1 → Hospital2.
- DICOM servers (Orthanc / dcm4chee) both sides point to local bucket; DB metadata replicates via Postgres.
- Nightly reconciler: `DICOM object without DB row` and vice versa → quarantine list.
- New studies during failover: write to promoted DC bucket; backfill on failback.

Bandwidth sizing: `daily PACS GB * 8 / replication_window_sec * overhead(1.5)`. Example: 200 GB/day over 8h → ~83 Mbps + headroom → provision ≥150 Mbps.

## 3. Files / Shares

- `rsync --archive --delete` or Syncthing/SeaweedFS replication every 5-15 min for Tier-1 shares; hourly for Tier-2.
- Keep versioned snapshots (ZFS/btrfs or VSS) 14 days to survive ransomware + accidental delete.

## 4. VMs / Containers

- Prefer container/K8s stateless + external DB/storage (failover = scale up DR deployment, no VM copy).
- For legacy VMs: ZFS/zvol or storage-level snapshots shipped every 15-60 min (e.g., `syncoid`), or Veeam/Zerto replica.
- Golden images in both DCs; config via Ansible (no snowflake servers).

## 5. Direction: Why not true bidirectional?

True multi-leader causes write conflicts in clinical orders. Use:

- Normal: H1 → H2 one-way.
- Failover: promote H2 (now H2 → H1 when H1 returns as replica).
- Failback: planned switchover, reverse direction.

This gives "<->" capability without conflict.

## 6. Backup (separate from replication!)

3-2-1-1-0: 3 copies, 2 media, 1 offsite (other DC), 1 immutable/offline, 0 errors (tested restore).
- Full weekly + incr daily, WAL archiving continuous.
- Monthly restore test + quarterly DR drill. Log evidence for audit.
- Replication is NOT backup – ransomware/deletes replicate too. Immutable snapshots required.
