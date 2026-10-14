# Roadmap

## Phase 0 – Decide (1-2 weeks)
- [ ] Sign off BIA tiers + RTO/RPO targets with directors + clinicians.
- [ ] Measure PACS GB/day, DB WAL/day → size links + storage.
- [ ] Choose witness location. Lower DNS TTL plan. Paper-downtime SOP refresh.

## Phase 1 – Visibility (2-4 weeks)
- [ ] Deploy bcms-agent both DCs + Prometheus/Grafana + lag alerts.
- [ ] Nightly backup + monthly restore test. Start audit log.

## Phase 2 – Data Safety (4-8 weeks)
- [ ] Postgres H1→H2 streaming + PITR archiving. Failover script tested in staging.
- [ ] MinIO/Ceph mirror + PACS reconciler. File-share sync.

## Phase 3 – Failover Ready (4-6 weeks)
- [ ] K8s/VM DR manifests scaled-min in H2. DNS/LB cutover playbook.
- [ ] First tabletop + partial drill (one Tier-1 app). Measure RTO/RPO.

## Phase 4 – Compliance + Hardening (ongoing)
- [ ] Dashboard audit/drill export. Immutable backups. Ransomware isolate drill.
- [ ] Quarterly drills, yearly full-DC drill. BIA review.

## Acceptance Criteria (v1 done)
- Lag dashboard live, alerts paging.
- Documented failover <1h Tier-1 in drill with measured loss <= RPO.
- Restore test + drill packet exported and signed.
