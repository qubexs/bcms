# Tech Stack (all involved)

Covers dashboard, services, enterprise HIS integration, and infra. Start thin, expand.

## A. Web Dashboard – Node + React
- `apps/bcms-dashboard/` – React (Vite) + TypeScript + Tailwind.
- Backend `apps/bcms-api/` – Node (NestJS or Express) – auth, DR state machine, audit log, drill reports.
- DB for BCMS itself: Postgres (small, replicated too) + Prisma.
- Why: fastest for ops dashboard + audit UI clinicians/auditors use.

## B. Monitoring/Replication Helpers – Python + FastAPI
- `services/bcms-agent/` – FastAPI: collects lag, LSN, storage queue, exposes /healthz /metrics, heartbeats.
- `services/reconciler/` – Python jobs: PACS DICOM↔DB orphan check, order-count reconcile, bandwidth estimator.
- Why: best for scripting DB/storage checks + data jobs.

## C. Enterprise Clinical Services – Java Spring Boot
- `services/his-emr-api/` (template) – Spring Boot + FHIR R4 (HAPI FHIR), JPA/Hibernate → Postgres.
- Auth via Keycloak/OIDC + AD bridge; audit via Hibernate Envers.
- Why: hospital HIS/EMR vendors commonly Java; FHIR interop + BPJS bridging fit Spring ecosystem.

## D. Infra as Code (both DCs identical)
- `infra/terraform/` – per-DC: network, VMs/K8s cluster, Postgres (Stolon/CloudNativePG), MinIO/Ceph, LB.
  - `modules/` shared, `envs/hospital1/`, `envs/hospital2/`, `envs/witness/`.
- `infra/ansible/` – golden-image config, Patroni, Orthanc, Keycloak replica, bcms-agent deploy, DNS/LB cutover playbooks.
- `infra/k8s/` – Helm/Kustomize: app Deployments (HPA), Postgres operator, MinIO tenant, Prometheus/Grafana, Alertmanager.
- `infra/scripts/` – `failover.sh`, `failback.sh`, `drill-export.sh` (wrap runbook §2/§4).

## Proposed Repo Layout
```
apps/bcms-dashboard/  apps/bcms-api/
services/bcms-agent/  services/reconciler/  services/his-emr-api/
infra/terraform/ infra/ansible/ infra/k8s/ infra/scripts/
docs/ drills/
```

## Build Order (thin slice first)
1. BCMS agent + Prometheus/Grafana + lag dashboard (proves visibility).
2. Postgres streaming H1→H2 + failover script (proves RPO/RTO).
3. MinIO mirror + PACS reconciler (proves imaging).
4. Dashboard + audit/drill export (proves compliance).
5. Spring Boot FHIR template + Keycloak (proves clinical integration).
