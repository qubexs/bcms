# Security + Data Safety: Transaction & Transport (H1 <-> H2)

This doc makes §7 of ARCHITECTURE.md enforceable. Two guarantees:

1. **Transport:** every byte H1↔H2 is authenticated, encrypted, replay-safe.
2. **Transaction:** every clinical write is atomic, ordered, idempotent, and survives DC loss within RPO.

## 1. Threat Model

| Threat | Mitigation |
|---|---|
| MITM / sniffing on MPLS/Internet | IPsec + TLS everywhere, no plaintext replication ports |
| Replay (re-send old WAL/DICOM/order) | mTLS + nonces/LSN monotonicity, idempotency keys, WAL timeline checks |
| Split-brain double-write | Single-leader + quorum/witness + fencing; receivers reject writes from non-leader term |
| Ransomware / malicious delete replicating | Immutable backups + Object Lock + delayed-delete + anomaly halt |
| Insider / stolen creds | Short-lived certs + RBAC + 4-eyes for failover, break-glass audited |
| Corruption in flight/at rest | TLS + WAL CRC + object checksums (SHA-256) + nightly reconciler |
| Key loss | KMS/HSM + sealed backup of KEK, dual-custody restore |

## 2. Transport Security (H1 <-> H2)

### 2.1 Layers (defense in depth)
```
[App mTLS: service-to-service, FHIR API, bcms-agent]  <- SPIFFE/cert-manager, rotate ≤90d
[DB/Storage TLS: Postgres sslmode=verify-full, MinIO TLS, rsync over SSH]  <- private CA per BCMS
[Network IPsec: SD-WAN/IPsec or WireGuard tunnels, MPLS treated as untrusted]  <- PSK+pubkey, PFS
```

Rules:
- **No plaintext inter-DC ports.** Firewall default-deny; allowlist only: 5432 (PG, TLS-only), 9000 (MinIO TLS), 6443/443 (K8s/API mTLS), 51820/500/4500 (tunnels), 9100-series metrics scraped via TLS.
- **TLS ≥1.2, prefer 1.3.** Ciphers: `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY_SHA256`; ECDHE+AESGCM for 1.2. Disable SSLv3/TLS1.0/1.1, RC4, DES, NULL.
- **mTLS for service calls** (FHIR, agent→orchestrator, reconciler→DB/storage). Each DC has identity `spiffe://bcms/h1/*`, `spiffe://bcms/h2/*`. Verify SAN, not just IP.
- **Two links, both encrypted:** MPLS gets IPsec overlay too (don't trust carrier). Failover between links must re-establish SA, not fall back to plaintext.

### 2.2 PKI / Certs
- Private CA per BCMS (Vault PKI or step-ca), offline root + online intermediate per DC.
- Issuance via cert-manager (K8s) / Vault agent (VM). Lifetimes: workload 30-90d auto-renew, DB/server 90d, root 5y.
- Revocation: OCSP/CRL + short TTL; compromised DC cert → revoke + rotate intermediate.
- Store: private keys in HSM/KMS or sealed Vault; never in git. Backup KEK with Shamir 2-of-3 custodians.

### 2.3 Reference Configs

Postgres (both DCs, `postgresql.conf` + `pg_hba.conf`):
```conf
ssl = on
ssl_cert_file = '/etc/pki/bcms/db-server.crt'
ssl_key_file  = '/etc/pki/bcms/db-server.key'
ssl_ca_file   = '/etc/pki/bcms/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384'
# require client cert for replication
```
```conf
# pg_hba.conf – replication only with cert + IP allowlist
hostssl replication replicator 10.20.0.0/16 cert clientcert=verify-full
hostssl all         appuser    10.20.0.0/16 cert clientcert=verify-full
host    all         all        0.0.0.0/0    reject
```
Primary conninfo (standby):
```
primary_conninfo = 'host=h1-db port=5432 sslmode=verify-full sslrootcert=/etc/pki/bcms/ca.crt sslcert=/etc/pki/bcms/db-repl.crt sslkey=/etc/pki/bcms/db-repl.key'
```

MinIO bucket replication (TLS + SSE-KMS):
```bash
mc alias set h1 https://minio.h1:9000 SVC_H1 --api S3v4
mc alias set h2 https://minio.h2:9000 SVC_H2 --api S3v4
mc admin kms key create h1 bcms-pacs-key
mc replicate add h1/pacs --remote-bucket https://minio.h2:9000/pacs \
  --remote-access-key SVC_H2 --replicate "delete,delete-marker,metadata" --bandwidth 100M
mc encrypt set sse-kms bcms-pacs-key h1/pacs h2/pacs
```

Spring Boot FHIR (mTLS out to H2 / BPJS bridge):
```properties
server.ssl.enabled=true
server.ssl.protocol=TLS
server.ssl.ciphers=TLS_AES_256_GCM_SHA384,TLS_AES_128_GCM_SHA256
server.ssl.client-auth=need
server.ssl.key-store=file:/etc/pki/bcms/svc.p12
server.ssl.key-store-type=PKCS12
server.ssl.trust-store=file:/etc/pki/bcms/ca.p12
```

FastAPI bcms-agent (TLS + token):
```python
# uvicorn --ssl-certfile /etc/pki/bcms/agent.crt --ssl-keyfile /etc/pki/bcms/agent.key
from fastapi import Depends, FastAPI, Header, HTTPException
app = FastAPI()
def require_svc(auth: str = Header("")):
    # mTLS verified at proxy; plus short-lived bearer for defense in depth
    if auth != f"Bearer {os.environ['AGENT_TOKEN']}": raise HTTPException(401)
```

WireGuard example (secondary link):
```ini
[Peer] # H2 on H1
PublicKey = <H2-pub>
AllowedIPs = 10.200.0.2/32
Endpoint = h2.example:51820
PersistentKeepalive = 25
# + firewall: accept 51820/udp only from peer IPs
```

## 3. Transaction & Data Safety

### 3.1 Golden Rules
1. **Single-leader writes.** Only PRIMARY DC accepts clinical writes. DR API returns `409 + Retry-After + primary Hint` for writes (reads allowed from replica with `Warning: read-from-standby` header).
2. **DB is the commit point.** PACS objects/files are committed only after DB tx commits (or via outbox – never the reverse).
3. **Every cross-DC message is idempotent.** Require `Idempotency-Key` (UUIDv7) on orders/results; receivers dedupe on `(key)` with 7-day window.
4. **Order by LSN, reject gaps.** Replicator/reconciler tracks `last_applied_lsn`; gaps → halt + page, never skip.
5. **Bounded loss = RPO.** In-flight tx at failover second is either replayed from WAL/archive or declared lost in audit with patient-reconciliation SOP.

### 3.2 Sync vs Async (what to use)
- **Intra-DC:** sync (`synchronous_commit = remote_apply`, 1 local standby) → zero-loss HA inside DC.
- **Inter-DC (H1→H2):** async streaming + WAL archiving (default). If RTT <50ms and bandwidth headroom: `synchronous_commit = remote_write` with H2 as 2nd sync standby for Tier-1 only. Otherwise async + 15-min RPO enforced by lag alerts.
- Never use `synchronous_commit = off` for Tier-1.

### 3.3 Outbox Pattern (required for DB+storage/API fan-out)
Prevents "DB committed but PACS/API not sent" and double-sends:

```sql
CREATE TABLE outbox (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  agg_type text NOT NULL, agg_id text NOT NULL,
  type text NOT NULL, payload jsonb NOT NULL,
  idempotency_key uuid NOT NULL UNIQUE,
  created_at timestamptz DEFAULT now(), sent_at timestamptz,
  lsn pg_lsn
);
-- app tx: INSERT business row + INSERT outbox in SAME tx
-- relay: SELECT ... FOR UPDATE SKIP LOCKED -> publish -> UPDATE sent_at
```

Relay publishes to: MinIO metadata queue / FHIR bundle sender / BPJS bridge / audit log. Consumers dedupe on `idempotency_key`. Relay is the only cross-DC sender; app never calls H2 directly in tx path.

Sequence for new lab order:
```
1. App (H1): BEGIN; INSERT orders; INSERT outbox(order.created); COMMIT -> returns order_id + Idempotency-Key
2. Relay: read outbox (ordered by LSN) -> POST https://h2/sync (mTLS) with key
3. H2 receiver: INSERT ... ON CONFLICT (idempotency_key) DO NOTHING -> 200 {applied: true/false}
4. Relay: mark sent_at. Lag metric = now - min(created_at WHERE sent_at IS NULL)
```

### 3.4 FHIR / HIS API Safety
- Use FHIR transaction Bundles (`Bundle.type=transaction`) for multi-resource writes; server validates + commits atomically, returns `Bundle` with per-entry status.
- `If-None-Exist` + conditional creates for dedupe; `ETag`/`If-Match` (version-aware update) to prevent lost updates.
- Every response echoes `Idempotency-Key` + `X-Primary-DC: h1` + `X-Applied-LSN`.
- DR (standby) behavior: `GET` ok (with staleness header `X-Replica-Lag-Sec`), `POST/PUT/DELETE` → `409 Conflict {error: "not-primary", primary: "https://h1"}` unless promoted.

### 3.5 PACS Two-Phase (object + metadata)
1. Upload DICOM bytes to H1 bucket with `Content-SHA256` + `x-amz-meta-study-uid`, `x-amz-meta-idempotency-key`.
2. Commit DB row (study/instance UID unique) + outbox entry in same tx.
3. Relay confirms object exists (HEAD + checksum) then marks sent; H2 mirror validates checksum on replicate.
4. Reconciler (nightly + post-failover): lists `objects without DB` → quarantine bucket; `DB without objects` → re-queue upload / flag for modality resend. No auto-delete.

### 3.6 Integrity Everywhere
- At rest: LUKS/ZFS native encryption for DB volumes, Ceph/MinIO SSE-KMS (per-bucket keys), KMS backed by HSM/Vault.
- In transit: §2 (TLS+IPsec). Checksums: Postgres `data_checksums=on` + WAL CRC, S3 SHA-256, rsync `--checksum`, DICOM SOP Instance UID uniqueness.
- Immutable recovery: S3 Object Lock (Compliance, 30-90d) on backups + WAL archive; ZFS snapshots `hold`; delayed bucket-delete (e.g., replicate deletes after 24h) so ransomware deletes don't instantly wipe DR.

### 3.7 Failover Data-Safety Checklist (with tx guarantees)
- [ ] Record `last_received_lsn` on H2 + `last_archived_wal`; compute loss window = primary LSN − replica LSN.
- [ ] Promote H2 (`pg_ctl promote`), verify timeline increments; old H1 fenced (LB drain + `pg_ctl stop -m fast` remotely if reachable, revoke certs if compromised).
- [ ] Drain outbox: relay replays unsent (ordered) to new primary; receivers dedupe.
- [ ] PACS reconcile: run orphan check before opening writes; resend missing from modalities/WAL archive.
- [ ] Open writes with new term/epoch (`primary_term++`); clients discard cached `X-Primary-DC: h1` routes.
- [ ] Downtime-window paper orders re-entered with new idempotency keys + `reconciled:true` flag + dual nurse verify.
- [ ] Scribe logs: LSN gap, lost-tx list (from WAL gap), re-entry count. Sign-off by Commander+Verifier.

## 4. AuthN/Z + Audit (cross-DC)

- Humans: Keycloak OIDC + AD bridge, MFA required for Commander/Executor; break-glass local accounts sealed, every use pages + auto-expires 8h.
- Services: mTLS SPIFFE IDs + short-lived JWT (5 min) with `aud: h2-sync`, `scope: replicate:write`.
- RBAC (4-eyes): `declare-disaster` ≠ `execute-failover` roles; single user cannot hold both in same incident.
- Audit: append-only log (WORM bucket) – every declare/promote/rollback, cert rotation, break-glass, key use. Ship to both DCs + witness. Format: `{ts, actor, role, action, term, lsn, result, hash_prev}` (hash-chained).

## 5. Compliance Mapping (Indonesia hospital)

- UU Kesehatan + Permenkes RME (rekam medis elektronik): integrity, access control, audit trail, retention – covered by §3-4 + drill packets.
- BPJS bridging: TLS + idempotent claim submission (claim number as idempotency key), no duplicate billing after failover.
- ISO 27001 A.8/A.10/A.13/A.17: encryption, key mgmt, network controls, continuity – map each control to this doc in audit packet.
- PDP Law (UU No. 27/2022): patient PII encrypted at rest/in transit,最小 access, breach notification sourced from audit log.

## 6. Validation (prove it)

- Weekly: `pg_verify_checksums` sample + MinIO `mc admin heal` + cert-expiry check (<14d alert).
- Monthly: restore from WAL archive to PITR point; chaos: kill one link (must stay encrypted, lag recovers); replay old message (must dedupe, no double order).
- Quarterly drill: measure actual RPO (LSN gap at promote) + list lost-tx + paper reconcile count. Export drill packet PDF.
- Yearly: full TLS/PKI rotation rehearsal + HSM restore rehearsal.

## 7. Minimal Do-Now Checklist

- [ ] Enable PG `ssl=on` + `hostssl ... cert` + `data_checksums=on`; MinIO TLS + SSE-KMS; IPsec on both links.
- [ ] Add `outbox` table + relay + `Idempotency-Key` on order/result APIs; DR returns 409 for writes.
- [ ] Object Lock on backup/WAL buckets + 24h delayed-delete on PACS mirror.
- [ ] Vault/step-ca + cert-manager; rotate all static passwords to short-lived certs/JWT.
- [ ] Turn on hash-chained audit log shipped to H2 + witness.
- [ ] Run replay + link-kill + PITR tests; file results in `drills/`.
