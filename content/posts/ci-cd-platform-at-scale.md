---
title: "Building a Centralized CI/CD Platform for Microservices"
date: 2024-09-15
description: "Lessons from centralizing CI/CD workflows into a reusable platform that handles build, test, deploy, and rollback across a microservices organization."
tags: ["GitHub Actions", "Helm", "SLSA", "Kubernetes"]
---

When you have many backend microservices, each with its own CI/CD pipeline, things get messy fast. Duplicated workflows, inconsistent deploy practices, no governance — it's a maintenance nightmare that slows everyone down.

A centralized CI/CD platform solves this: a single repository of reusable workflows that every service consumes. Here's what I've learned building one.

## The Problem with Distributed Pipelines

When every service has its own `.github/workflows/` with copy-pasted workflow files, updating the deploy process means opening PRs across dozens of repositories. Changes are inconsistent, some repos fall behind, and there's no single source of truth for "how we deploy."

## The Reusable Workflows Approach

GitHub Actions' `workflow_call` trigger lets you define workflows in a central repository and call them from any other repo:

```yaml
# In any service repo — one line for the full pipeline
jobs:
  deploy:
    uses: org/platform/.github/workflows/deploy.yml@main
    with:
      service-name: my-service
      environment: production
    secrets: inherit
```

One reference in your service repo, and you get the full deploy pipeline — build, scan, push, helm upgrade, health check, rollback on failure.

## Helm Governance

Standardizing Helm charts with a governance model is critical:

- **Base chart** — shared across all services, maintained centrally
- **Service overrides** — minimal per-service values files
- **Validation** — `helm template` runs in CI before any deploy
- **Rollback safety** — automatic rollback if health checks fail post-deploy

The tricky part is subchart value scoping — if your base chart uses nested subcharts, values don't automatically propagate. Using `global.*` keys solves this but requires careful documentation.

## Supply Chain Security Basics

Every container image should go through:

- **Vulnerability scanning** — CVE checks before pushing to registry
- **SLSA provenance** — signed build attestations
- **OIDC authentication** — no long-lived secrets, cloud auth via short-lived tokens
- **SHA pinning** — all third-party action versions pinned to exact commit SHAs

## Lessons Learned

The hardest part isn't the technical implementation — it's **adoption**:

1. **Don't force migration** — make the platform so good that teams want to switch
2. **Maintain backwards compatibility** — old workflows should still work during transition
3. **Document everything** — comprehensive docs are as important as the code itself
4. **Measure impact** — DORA metrics give objective proof the platform is working

Building a CI/CD platform is building a product. Your developers are your users. Treat it accordingly.
