# GitOps Deployment Platform

![architecture write-up](https://img.shields.io/badge/type-architecture%20write--up-blue) ![code](https://img.shields.io/badge/code-private%20(employer)-lightgrey) ![stack](https://img.shields.io/badge/stack-ArgoCD%20%C2%B7%20Helm%20%C2%B7%20Kubernetes-informational)

> **Architecture write-up.** This documents a system I designed and built as a DevOps Engineer at Faclon Labs (Mumbai). The implementation belongs to my employer and is not published; what follows is the design, the moving parts and the outcomes, with internal names and details left out.

Kubernetes deployments standardised on Helm charts and ArgoCD: the desired
state of every environment lives in Git, ArgoCD keeps the clusters in sync
with it, every release is auditable, and rolling back is a one-command
revert.

## The problem

Deployments were imperative — someone running `kubectl`/`helm` from a
laptop. What was running could differ from what was in Git, and rolling back
depended on remembering what the previous state had been.

## Architecture

```mermaid
flowchart LR
    D[Developer] -->|merge PR| G[(Git<br/>app config repo)]
    CI[CI pipeline] -->|push image| REG[(Container registry)]
    CI -->|bump image tag| G
    A[ArgoCD] -->|watches| G
    A -->|renders| H[Helm charts]
    A -->|sync| K1[Cluster · staging]
    A -->|sync| K2[Cluster · production]
    K1 & K2 -->|pull| REG
    A -->|drift / health| OBS[Dashboards & alerts]
```

## How a release happens

1. CI builds and pushes an image, then commits the new tag to the config repo.
2. ArgoCD notices the commit, renders the Helm chart with that environment's
   values and compares it with the live cluster.
3. The difference is applied; health checks decide when the release is done.
4. A rollback is `git revert` — ArgoCD syncs the cluster back to the previous
   state.

## The chart estate

Five Helm charts cover the data-science platform: the expression engine, its
on-demand worker pool (with AWS and Azure variants, each carrying its KEDA
ScaledObject), and the API gateway with its alerts handler. Each has on-prem
values for customer installs, and the production CI pipeline commits image-tag
bumps straight into the repository, so the chart history is the release
history. HTTPS termination moved from the nginx ingress to the cloud load
balancer, and hostnames, virtual hosts and queue names are values, not code.

## Design points

- **One chart, per-environment values** — the same artefact promotes from
  staging to production; only values differ.
- **Git is the audit log.** Who changed what, when, and the review that
  approved it, are the repository history.
- **Drift is visible**, and corrected automatically where that is safe.

## Outcomes

- Auditable, repeatable releases across environments.
- Automated synchronisation removed hand-run deployments.
- One-command rollbacks.

## Stack

ArgoCD · Helm · Kubernetes (EKS / AKS) · Git · container registry · Prometheus / Grafana

## My role

Introduced ArgoCD and Helm as the deployment layer, wrote the charts and
environment structure, and migrated the services onto it.
