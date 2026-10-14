# Business Continuity Management System (BCMS)
## HOSPITAL 1 (Data Center) <-> HOSPITAL 2 (Data Center)

Active-Passive with Bidirectional Sync capability for hospital-critical systems.

### Goal
Ensure hospital operations survive DC failure with:
- **RPO <= 15 min** (Tier-1), **<= 4h** (Tier-2)
- **RTO <= 1h** (Tier-1), **<= 8h** (Tier-2)
- No single-DC data loss for EMR/HIS/PACS
- Auditable DR drills + compliance docs

### Docs
- `docs/ARCHITECTURE.md` – full system design
- `docs/BIA-RTO-RPO.md` – Business Impact Analysis
- `docs/REPLICATION.md` – data replication design
- `docs/DR-RUNBOOK.md` – failover / failback SOP
- `docs/MONITORING.md` – health checks, alerts, dashboard
- `docs/TECH-STACK.md` – all-involved stack (React + Node, Python FastAPI, Java Spring Boot, Infra as Code)
- `docs/SECURITY-TRANSPORT.md` – transaction + transport safety (mTLS/IPsec, outbox, idempotency, keys, audit)
- `docs/ROADMAP.md` – phased implementation

### Topology (logical)
```
              +----------------+  MPLS/SD-WAN + IPsec  +----------------+
              | HOSPITAL 1 DC  |<------------------------>| HOSPITAL 2 DC  |
              |  PRIMARY       |   replication + quorum  |  STANDBY/DR    |
              +-------+--------+                         +-------+--------+
                      |                                          |
           [HIS/EMR/PACS/LIS/Pharmacy/Billing/AD/PACS-storage]  [replica cluster]
                      |                                          |
                 [BCMS Agent + Monitor] <------quorum/witness------> [BCMS Agent + Monitor]
```

> Rule #1: No automatic split-brain failover without quorum/witness. Prefer manual declare + automated execute.

See `docs/ARCHITECTURE.md` for detail.
