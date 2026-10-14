# DR Runbook – Failover / Failback / Drill

## Roles
- Commander (declares disaster), Executor (runs playbook), Verifier (clinical smoke test), Scribe (logs).

## 0. Preconditions (both DCs, always green before drill)
- [ ] Replication lag < 60s Tier-1
- [ ] Last backup restore-test passed
- [ ] DR apps deployable (helm/ansible dry-run ok)
- [ ] DNS TTL lowered to 60s
- [ ] Witness reachable

## 1. Declare (manual, 2-person rule)
1. Commander confirms: DC-1 down >10 min or data-risk (fire/ransomware) + witness agrees DC-1 unreachable.
2. Scribe records: time, lag at declare (`pg_stat_replication`), reason, approvers.
3. Announce: ER/ICU downtime procedure (paper fallback) ON.

## 2. Execute Failover to Hospital 2 (target <60 min)
1. Fence old primary: block replication + LB traffic to DC-1 (prevent split-brain if it comes back).
2. Promote DB: `patronictl failover` / `pg_ctl promote` on DC-2 replica. Record WAL LSN + data-loss window.
3. Promote storage: bucket/shares to RW in DC-2.
4. Scale apps: `kubectl scale` / compose up DR profile; point to local DB/storage/auth replica.
5. Promote auth: Keycloak/AD to RW.
6. Switch traffic: DNS/LB to DC-2 endpoints.
7. Verify (Verifier):
   - Login, admit test patient, order lab + med, view last CT in PACS, print label, BPJS bridge ping.
   - Reconciler: count last-hour orders in DC-2 vs pre-failover LSN.
8. Announce: digital ops resumed on DC-2; keep paper reconciliation for downtime window.

## 3. Operate on DR (hours–days)
- Monitor lag (now DC-2 primary, DC-1 absent → WAL archiving only).
- Increase backup frequency. Freeze non-urgent changes.

## 4. Failback to Hospital 1 (planned)
1. Repair/audit DC-1, wipe-or-trust decision (if ransomware: rebuild from gold).
2. Rebuild DC-1 as replica: `pg_basebackup` / `pg_rewind` from DC-2 + resync buckets (mirror reverse).
3. Verify lag ~0, then scheduled switchover (reverse of §2) in low-load window.
4. Keep DC-2 as new DR replica. Post-mortem within 72h.

## 5. Drill cadence
- Quarterly: tabletop + technical failover of one Tier-1 app (no patient impact window).
- Yearly: full-DC failover drill + timed RTO/RPO measurement.
- Every drill produces: lag graph, RTO measured, issues, CAPA list. Store in `drills/YYYY-MM-DD/`.

## 6. Anti-patterns
- Never auto-promote without quorum.
- Never bring old primary online RW without re-sync.
- Never skip clinical smoke test (IT-green ≠ care-ready).
