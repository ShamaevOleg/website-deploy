# website-deploy

GitOps repository for deploying the REMNI application to EKS with Argo CD.

This repository holds the **desired deploy state** — Helm charts and Argo CD
Application definitions — kept deliberately apart from two other repositories:
the private application code (its own repo) and the infrastructure
(`aws-devops-learning`). Each has a different lifecycle and owner; keeping them
separate keeps every change reviewable on its own and keeps private code out of
the public deployment history.

Argo CD runs inside the cluster, watches this repository, and reconciles the
cluster to match it. Deployment happens by git commit, not by `kubectl apply`.

## Layout

| Path | What's here |
|---|---|
| `charts/backend/` | The Django backend: Deployment, Service, ConfigMap, a migration Job ordered by Argo sync-waves, probes, and a non-root security context. Reads its database password from a Kubernetes Secret synced out of AWS Secrets Manager. |
| `charts/frontend/` | The nginx frontend (unprivileged, port 8080): serves the SPA and reverse-proxies `/api`, `/admin`, `/media`, `/static` to the backend inside the cluster. |
| `charts/redis/` | A minimal in-cluster Redis for a shared cache (a stand simplification — production would use ElastiCache). |
| `external-secrets/` | The ClusterSecretStore and ExternalSecret that pull the RDS password from Secrets Manager into a Kubernetes Secret via the External Secrets Operator. |
| `apps/` | Argo CD Application definitions — one per component — pointing at the charts above. |
| `ingress/` | The ALB Ingress that exposes the frontend, with TLS terminated on the load balancer via an ACM certificate. |

## How a deploy happens

1. The application pipeline (in the private app repo) builds an image, pushes it
   to ECR, and updates the image tag in the relevant chart's `values.yaml` here.
2. Argo CD sees the commit and syncs the change into the cluster.
3. Sync-waves order the rollout: config and secrets first, the migration Job
   next, then the application pods — so the database schema is migrated before
   new pods serve traffic.

## Notes

Secrets never live in this repository. The database password is delivered to
pods at runtime from AWS Secrets Manager through the External Secrets Operator,
authenticated by IRSA — there are no credentials in git.

Bring-up and teardown of the whole stack are documented in `runbook.md`.
The infrastructure this deploys onto lives in the `aws-devops-learning`
repository.
