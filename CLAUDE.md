# CLAUDE.md — CaaS Portal

Project context and working agreement for any AI coding agent (Claude Code / Cowork)
working in this repository. Read this fully before editing anything.

## What this is

A self-service **Cluster-as-a-Service portal**. Authenticated users fill out a form and get a
Kubernetes cluster provisioned into **their own OpenStack project** via a GitOps pipeline built on
Cluster API (CAPI), the OpenStack provider (CAPO), Flux CD, and Sealed Secrets. The portal drives a
forked copy of `stackhpc/capi-helm-fluxcd-config`.

## Two invariants — never violate these

1. **One persistent management cluster.** The portal talks to a single long-lived management
   cluster that runs CAPI + CAPO + Flux + sealed-secrets + the portal backend. Tenant _workload_
   clusters are provisioned into tenant OpenStack projects, but their control resources (`Cluster`,
   `OpenStackCluster`, `HelmRelease`, `<name>-kubeconfig` secret) live on the management cluster.
   Do **not** reintroduce the upstream per-cluster `kind` bootstrap model for tenant clusters.

2. **OpenStack credentials are sealed, never stored in plaintext.** The application-credential
   secret is sealed with `kubeseal` the instant it is received, committed to git only as a
   `SealedSecret`, and the plaintext is dropped immediately after. Never write plaintext credentials
   to disk in the repo checkout, never log them, never return them from any API, never persist a
   readable copy in Postgres. The `credentials.enc_secret` column is optional envelope-encrypted
   bytes for re-sealing only and should default to NULL.

## Tech stack (do not swap without discussion)

- Backend: Python 3.12, FastAPI (async), SQLAlchemy 2.0 + Alembic, Celery + Redis for jobs.
- Data: PostgreSQL 16.
- K8s/Git/OpenStack clients: `kubernetes`, `pygit2`/GitPython, `openstacksdk`.
- Sealing: `kubeseal` CLI against the in-cluster sealed-secrets controller.
- Auth: OIDC via Authlib. Providers: Google, MyAccessID.
- Frontend: React + TypeScript + Vite, Tailwind CSS, TanStack Query.

## Repository map

```
backend/            FastAPI app: routers, models, tasks, services
  models.py         SQLAlchemy models (users, projects, memberships, credentials, clusters)
  routers/          auth, options, clusters
  tasks/            Celery: provision, poll, scale, delete
  services/         git, seal, manifests, openstack, k8s
frontend/           React + Tailwind
config-repo/        the forked capi-helm-fluxcd-config (git submodule or separate checkout)
  clusters/<tenant-slug>/<cluster-name>/  per-cluster dir the backend writes
.claude/skills/     project skills (see below)
```

## Phased delivery — build strictly in order

**Phase 0 — Foundations (no portal code).**
Verify OpenStack prerequisites (project + quota, application credentials, Octavia LoadBalancer,
external network for floating IPs, DNS). Stand up the management cluster with CAPI, CAPO, Flux,
sealed-secrets, cert-manager, the `capi-helm` HelmRepository, and a Flux `Kustomization` over
`./clusters` with `prune: true`. Fork the config repo. **Provision one workload cluster by hand**
following `clusters/example` to prove the pipeline end to end. Nothing else starts until this works.

**Phase 1 — Read-only portal.**
FastAPI skeleton, Postgres + Alembic, the models, OIDC with **one** provider, the project/RBAC
model, and a dashboard that lists clusters by reading the management cluster + DB. No provisioning.

**Phase 2 — Provisioning.**
The create form, options service, the seal → render → commit → push worker, and the status poller
that advances rows through `Pending → Provisioning → Provisioned/Failed`.

**Phase 3 — Lifecycle actions.**
Kubeconfig download, worker scaling (patch `helmrelease.yaml` + push), deletion (remove dir + push,
Flux prunes).

**Phase 4 — Hardening & multi-tenant scale.**
Second OIDC provider (MyAccessID), envelope encryption for the optional at-rest copy, per-project
quota enforcement, observability, and an HA management cluster.

Each phase must be independently deployable and demoable before the next begins.

## Coding conventions

- Every cluster route is scoped by project membership. Never return or mutate a cluster the caller's
  `Membership` does not grant access to. RBAC roles: `viewer` (read), `operator` (create/scale/
  delete), `admin` (+ manage members).
- Cluster/secret/namespace names must be DNS-1123 valid. Validate before they reach git or k8s.
- Manifest value keys must match the `openstack-cluster` chart version pinned in the fork. Do not
  invent keys — check `config-repo` and the chart's `values.yaml`.
- Long-running work (provision/poll/scale/delete) goes in Celery tasks, never inline in a request.
- All secrets flow through the seal service; nothing else touches raw credential material.

## Commands

```
# backend
uvicorn backend.app:app --reload
alembic upgrade head
celery -A backend.tasks worker -l info
celery -A backend.tasks beat -l info      # status poller schedule
# frontend
npm --prefix frontend run dev
```

## Skills

Project skills live in `.claude/skills/`. Consult them for the tasks they cover:

- `capi-cluster-manifests` — rendering and validating a per-cluster config directory.
- `flux-gitops-workflow` — the seal / commit / push / reconcile / delete procedure and its safety rules.

## Do not

- Do not commit plaintext credentials or a `kubeconfig` to the config repo.
- Do not block a request thread on provisioning or OpenStack calls.
- Do not bypass the management-cluster model or the sealing service.
- Do not change the tech stack, schema, or phase order without updating this file first.
