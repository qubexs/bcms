# BCMS Architecture – Hospital 1 <-> Hospital 2

## 1. Objectives
1. Continuity of life-critical systems: HIS, EMR/EHR, PACS/RIS, LIS, Pharmacy, ICU/ER, Blood bank, Billing/BPJS bridge.
2. Survive scenarios: DC power loss, fire/flood, network cut, ransomware, storage/DB failure, regional disaster.
3. Auditable: BIA, risk register, DR drill logs, RTO/RPO evidence.

## 2. Model Decision: Active-Passive + Active-Active for stateless

- **Active-Passive (recommended default):** Hospital 1 = Primary, Hospital 2 = DR Standby (hot standby). All writes go to Primary, async/sync replicated.
- **Active-Active only for:** stateless apps (portal, mobile API, appointment), read-replica queries, DNS/GLB-routed.
- **No Active-Active writes** for EMR/HIS DB (avoids conflict/split-brain). If bidirectional needed, use conflict-free pattern: single-leader + staged promotion, not multi-leader.

Why: hospital data cannot tolerate merge conflicts (medication orders, lab results).

## 3. Layers

```
[Clients: HIS workstations, PACS modalities, Pharmacy, BPJS bridge]
   |
[Global Traffic: DNS failover + HAProxy/Nginx/Cloud LB, health-checked]
   |
[App Tier per DC: HIS-API, EMR-API, FHIR gateway, Auth (Keycloak/AD replica)]
   |
[Data Tier per DC: PostgreSQL cluster, PACS object storage, file shares]
   |
[Replication: WAL streaming / logical, rsync/MinIO mirror, VM/ZFS snapshots]
   |
[BCMS Control Plane: monitor agent, orchestrator, dashboard, audit log]
```

## 4. Components per DC

| Layer | Hospital 1 (Primary) | Hospital 2 (DR) |
|---|---|---|
| App | K8s/VM: his-api x3, emr-api x3, fhir-gw x2, auth x2 | Same, scaled min (x1-2), auto-scale on failover |
| DB | Postgres Primary + local sync standby | Postgres async replica (cascaded), promotable |
| PACS Imaging | Primary object store (MinIO/Ceph) + DICOM server (Orthanc/dcm4chee) | Mirrored bucket + standby DICOM |
| Files | SMB/NFS + versioned backup | Read-only replica, RW on promotion |
| Identity | AD/LDAP Primary + Keycloak Primary | AD replica + Keycloak replica (read-only until promotion) |
| BCMS Agent | heartbeat, checks, replication lag exporter | same |
| Backup | Immutable local + offsite copy | Independent backup (3-2-1-1-0 rule) |

Witness/quorum: 3rd small node (cloud VPS or HQ) to avoid split-brain. Only 1 vote, no data.

## 5. Network

- Dual links: MPLS primary + Internet SD-WAN/IPsec secondary. Minimum 100 Mbps dedicated for replication (size by PACS volume).
- Latency budget: <50ms RTT for sync WAL; if >50ms use async + synchronous_commit=remote_write or full async for Tier-1 with 15-min RPO.
- Segmentation: VLANs for clinical, imaging, management, replication (dedicated VLAN/qos). Firewall allowlist replication ports only.
- DNS failover: TTL 60s for `his.hospital`, `emr.hospital`, `pacs.hospital`. Or Global Server Load Balancing if available.

## 6. Data Consistency Rules

1. DB is source of truth. Storage follows DB transaction order where possible.
2. PACS: DICOM object + DB metadata must reconcile (nightly orphan check job).
3. Failover = promote DB replica → promote storage to RW → start apps → switch DNS/LB → verify.
4. Failback = re-sync old primary as replica (pg_rewind/rsync), then planned switchover. Never just turn old primary back on.

## 7. Security

- mTLS + IPsec for inter-DC. Encrypted replication (TLS), encrypted at rest (LUKS/Ceph encryption).
- Immutable backups (S3 Object Lock) against ransomware.
- RBAC: Declare-disaster vs Execute-failover are separate approvals (4-eyes).
- Audit every drill/failover: who, when, lag at promotion, data loss window, checklist.
- Full spec: `SECURITY-TRANSPORT.md` (TLS 1.2+, cipher suites, PKI rotation, pg_hba cert auth, MinIO SSE-KMS, outbox + Idempotency-Key, single-leader writes, hash-chained audit).

## 8. Failure Modes Handled

| Failure | Detection | Action |
|---|---|---|
| DB primary down, DC up | Patroni/Stolon health | Local auto-failover within DC |
| Full DC-1 down | BCMS monitor + witness agree | Promote DC-2 (manual declare, auto execute) |
| Link down (split-brain risk) | Quorum loss on one side | No auto-promote; primary stays if it holds quorum |
| Ransomware | Integrity/anomaly alert | Isolate, promote clean replica/snapshot, no failback to infected |
| PACS flood (storage full) | Capacity alert | Tier to DC-2, expire/purge per retention policy |

## 9. Non-Goals (v1)
- No multi-leader EMR writes.
- No zero-downtime guarantee for in-flight transactions (guarantee bounded loss <= RPO).
