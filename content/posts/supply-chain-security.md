---
title: "Supply Chain Security in Practice: From Container Scanning to Compliance"
date: 2024-06-20
description: "A practical guide to securing your CI/CD pipeline: container vulnerability scanning, secrets management, SHA pinning, and preparing for compliance audits."
tags: ["Security", "Compliance", "Secrets Management", "OIDC"]
---

Supply chain security sounds abstract until your CI pipeline gets compromised or an auditor asks how you manage secrets rotation. Here's how to make it practical.

## Start with What Hurts

Don't try to implement every SLSA level on day one. Start with the vulnerabilities most likely to cause damage:

1. **Compromised dependencies** — container images with known CVEs
2. **Leaked secrets** — long-lived credentials in CI/CD
3. **Untrusted actions** — third-party GitHub Actions running arbitrary code
4. **No audit trail** — no way to prove what was deployed and by whom

## Container Vulnerability Scanning

Integrate vulnerability scanning directly into the CI pipeline. Every container image should be scanned before it's pushed to the registry. High-severity CVEs should block the deploy.

One important detail: **scan the actual built image**, not a cached version. It's easy to end up in a situation where the scanner checks an older image tag while the pipeline deploys a newer, unscanned one. Always scan the local artifact.

## Secrets Management That Scales

For any compliance framework, you need to demonstrate controlled secrets management. A few principles that work well in practice:

- **Tag your secrets** with metadata — environment, owner, rotation schedule, sensitivity level
- **Enforce tagging via IaC** — non-compliant secrets can't be created if Terraform requires tags
- **Document rotation procedures** — both automated and manual rotation paths
- **Mind the consumer chain** — if services reference specific secret versions, you need to coordinate rotation with redeployments

The documentation serves double duty: it's both the operational runbook and the audit evidence.

## OIDC: Kill Your Long-Lived Credentials

Switching from static cloud access keys to **OIDC federation** is the biggest security win. CI/CD authenticates to cloud providers using short-lived tokens — no secrets to rotate, no keys to leak:

```yaml
permissions:
  id-token: write

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::role/deploy
      aws-region: eu-west-1
```

This single change eliminates an entire class of security risks.

## SHA Pinning for Actions

Every third-party GitHub Action should be pinned to an exact commit SHA, not a mutable tag:

```yaml
# Mutable tag — can be moved to point at malicious code
- uses: actions/checkout@v4

# Immutable SHA — safe from tag manipulation
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

More verbose, but it means a compromised action repository can't inject code into your pipelines. Tools like [zizmor](https://github.com/woodruffw/zizmor) can audit your workflows for unpinned actions automatically:

```text
$ zizmor .github/workflows/
warning[unpinned-uses]: action is not pinned to a hash
   --> deploy.yml:23:9
    |
 23 |     - uses: actions/checkout@v4
    |             ^^^^^^^^^^^^^^^^^^^ pin to a specific commit SHA

2 findings (1 warning, 1 note) across 1 file.
```

Wire it into pre-commit or CI and unpinned actions never make it past review.

## Making Compliance Practical

The trick is to **build compliance into your daily workflow** rather than treating it as a separate activity:

- Enforce policies in IaC — non-compliant resources can't be created
- Document procedures in code, not in spreadsheets
- Retain scanning results as CI artifacts — instant audit evidence
- Use OIDC to eliminate credential rotation burden entirely

When the audit comes, you're not scrambling to produce evidence. The evidence is your pipeline itself.

## Start Small, Iterate

You don't need a perfect supply chain security setup on day one. Start with:

1. Container scanning in CI (blocks obvious vulnerabilities)
2. OIDC for cloud auth (eliminates static credentials)
3. SHA pinning for actions (prevents supply chain attacks)

Then layer on SLSA provenance, formal secrets management, and compliance documentation as your needs grow. The important thing is to **start**.

## Related reading

- [Building a Centralized CI/CD Platform for Microservices](/posts/ci-cd-platform-at-scale/) — where to wire all this in
- [Observability as Code](/posts/observability-as-code/) — same pipeline, alerting and tracing
