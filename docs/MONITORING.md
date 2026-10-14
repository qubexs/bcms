# Monitoring + Alerting + Dashboard

## Signals (per DC + inter-DC)
- DB: `pg_stat_replication` replay_lag, WAL LSN gap, Patroni leader, archive success.
- Storage: bucket mirror lag/queue, disk %, DICOM reconciler orphans.
- App: /healthz per service, p95 latency, error rate, auth success.
- Net: RTT/loss H1↔H2, per-link status, DNS/LB endpoint health.
- BCMS: heartbeat age per agent, witness reachability, drill freshness (days since last drill).

## Alert Rules (example)
- Warning: replay_lag > 60s (Tier-1), link loss on one path, cert <14d.
- Critical/page: replay_lag > 5 min, leader loss, both links down, backup fail, reconciler orphans spike, heartbeat > 2 min.
- RPO-breach: lag > 15 min Tier-1 → page Commander.

## Stack (lightweight, all-FOSS)
- Exporters: postgres_exporter, node_exporter, blackbox_exporter (H1↔H2 probes), custom `bcms-agent` (/metrics: lag, LSN, queue).
- Store: Prometheus per DC (federate/cortex for global) + Loki for logs.
- Visual: Grafana dashboards: `BCMS-Overview`, `Replication-Lag`, `PACS-Reconcile`, `DR-Readiness`.
- Alerting: Alertmanager → WhatsApp/Telegram gateway + SMS + PagerDuty/Opsgenie; on-call rotation clinical-IT.

## Dashboard Must-Show
1. DC status: PRIMARY/DR, quorum holder.
2. Lag now per system + RPO budget bar.
3. RTO readiness checklist (% green).
4. Last drill date + measured RTO/RPO.
5. One-button drill packet export (PDF/CSV for audit).

## BCMS Agent (minimal contract)
`GET /healthz` → 200 + `{dc, role, lsn, lag_sec, queue}`; `GET /metrics` Prometheus format; heartbeat POST to orchestrator every 15s.
