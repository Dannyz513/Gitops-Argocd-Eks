# Design Document: GitOps Deployment with ArgoCD on EKS

## 1. Problem Statement

Every CI/CD pipeline in this portfolio so far has been **push-based**: a pipeline runs, authenticates to AWS, and pushes changes out. This works, but it has real drawbacks at scale — the pipeline needs broad credentials to reach the cluster, there's no single source of truth for what's *actually* running versus what was *intended* to run, and drift (someone running `kubectl apply` manually, bypassing CI) goes undetected.

**GitOps inverts this model.** An agent running inside the cluster continuously watches a Git repository and pulls changes, reconciling the live cluster state against what's declared in Git. Git becomes the single, auditable source of truth — not "whatever the last successful pipeline run happened to deploy."

## 2. Why Pull Instead of Push

**Security:** a push-based pipeline needs credentials capable of reaching into the cluster from outside — a real, standing external attack surface. A pull-based agent runs *inside* the cluster and only needs read access to a Git repo, a much smaller and more defensible permission boundary.

**Drift detection:** if someone manually changes a running resource (intentionally or not), a push-based pipeline has no way to know. A pull-based agent notices immediately, because it continuously compares live state against Git and flags (or auto-corrects) any difference.

**Auditability:** every change to what's running is a Git commit, with a real author, timestamp, and diff. Rolling back a bad deployment is `git revert`, not remembering which pipeline run to re-trigger.

## 3. Architecture

- **EKS cluster with Fargate** (reusing the proven pattern from the earlier EKS project), with the Fargate profile extended to cover the `argocd` namespace
- **ArgoCD**, installed via Helm, running inside the cluster
- **An ArgoCD Application resource** pointing at this same repository's `k8s-manifests/` folder — the repo is simultaneously the Terraform code, the Kubernetes manifests, and the documentation
- **Sync policy:** automated sync with self-healing enabled — meaning if someone manually changes a live resource, ArgoCD reverts it back to match Git automatically, actively enforcing Git as the source of truth rather than just detecting drift passively

## 4. What This Demonstrates

- Understanding of both major CI/CD paradigms (push, from earlier ECS/EC2 projects, and pull, here) and being able to articulate the real trade-offs between them
- Practical Argo CD / GitOps tooling experience, not just conceptual knowledge
- The same infrastructure (EKS + Fargate) applied to a different deployment paradigm than the earlier EKS project — showing the underlying Kubernetes skill transfers, not just one memorized workflow

## 5. Deliberately Out of Scope

- **Multi-cluster / multi-environment ArgoCD setups (App of Apps pattern)** — relevant at real organizational scale; this project demonstrates the core reconciliation loop on a single cluster
- **Sealed Secrets / external secrets operator** for managing sensitive values in Git safely — a real production GitOps setup needs this, since Git repos shouldn't contain plaintext secrets; noted here as a next step rather than implemented
- **Progressive delivery (canary/blue-green via Argo Rollouts)** — a natural extension once basic GitOps reconciliation is proven
