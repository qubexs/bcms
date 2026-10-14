# BIA + RTO/RPO – Hospital Systems

## Tier-1 (Critical, life/safety + revenue)
| System | Impact if down | RTO | RPO | Notes |
|---|---|---|---|---|
| HIS/EMR (ADT, ER, ICU, eRx) | Patient safety, cannot admit/triage | <= 1h | <= 15 min | Sync or near-sync replication |
| PACS/RIS (ER/CT) | Blocks radiology/ER | <= 1h | <= 15 min | Async object mirror + DB; accept short image lag, prioritize latest studies |
| LIS | Blocks labs, surgery | <= 2h | <= 15 min | Same as EMR |
| Pharmacy + Blood bank | Medication safety | <= 2h | <= 15 min | |
| Auth/AD + Single Sign-On | All apps blocked | <= 1h | <= 1h | Replica must be login-ready |
| Network/DNS/LB | Total outage | <= 30 min | n/a | |

## Tier-2 (Important)
| System | RTO | RPO |
|---|---|---|
| Billing / BPJS bridging / Insurance | <= 4h | <= 4h |
| Appointment / Queue / Mobile | <= 4h | <= 1h |
| HR/Finance/ERP | <= 8h | <= 4h |
| Email, file shares | <= 8h | <= 4h |

## Tier-3 (Deferrable)
Data warehouse, analytics, research extracts: RTO 24-72h, RPO 24h.

## BIA Method (minimal)
1. List services + dependencies (DB, AD, PACS storage, BPJS network).
2. Score: safety (1-5), financial (1-5), regulatory (1-5), reputational (1-5). Tier = max score >=4 → Tier-1.
3. Record MTPD (max tolerable period of disruption) = RTO upper bound.
4. Review quarterly + after every drill/incident.
