---
title: "Observability as Code: My Backend Meetup Talk, Now in Writing"
date: 2024-11-11
description: "A written adaptation of my Backend Meetup talk on observability in microservices — from the three pillars and OpenTelemetry to Terraform-managed alerts and on-call workflows."
tags: ["Observability", "OpenTelemetry", "Alerting", "Terraform", "Platform Engineering", "Meetup"]
---

A few weeks ago I gave a talk called **"Observability as Code: See It, Trace It, Fix It"** at our Backend Meetup at BetterMe. This post is a written version for anyone who couldn't make it — or just prefers reading over slides.

## Monitoring vs. Observability

The talk started with a distinction that sounds obvious but gets blurred in practice. **Monitoring** tells you when something is broken. **Observability** tells you _why_ — it lets you see relationships between services, identify root causes, understand how technical failures impact business metrics, and make complex systems predictable.

In a monolith, monitoring is relatively straightforward. In a microservices world, a single user request can cross dozens of services. Traditional monitoring can't give you the full picture — you need observability.

## The Three Pillars

Observability rests on three pillars, and each serves a different purpose:

**Traces** show the path of a request across microservices. When a checkout takes 8 seconds instead of 1, traces let you pinpoint which service in the chain is the bottleneck.

**Logs** capture detailed events and errors. They help you reproduce scenarios and understand the context behind an incident — what happened right before the failure, what parameters were involved.

**Metrics** — latency, error rate, throughput — provide quantitative insights into service health. They power dashboards and alerts, giving you the numbers you need to answer "is this service healthy right now?"

## OpenTelemetry as the Standard

A big chunk of the talk focused on **OpenTelemetry** — the emerging standard for instrumenting applications. The key selling point: it's vendor-neutral. You instrument your code once with OpenTelemetry, and you can send the data to whatever backend you choose — Elastic, Datadog, Jaeger, Grafana.

The architecture is straightforward: your application exports telemetry data through the OpenTelemetry SDK to a collector, which then routes it to your observability platform. The packages are available for most languages, and the integration is relatively lightweight.

## Distributed Tracing in Practice

This was the demo-heavy part of the talk. A span in distributed tracing typically includes:

- **Span name** — a descriptive label (e.g., `HTTP GET /users`)
- **Start and end timestamps** — the duration of the operation
- **Span context** — `trace_id`, `span_id`, and `parent_span_id` to link spans together
- **Attributes** — key-value pairs like HTTP status code, database query, or hostname
- **Events** — time-stamped annotations for significant moments within the span
- **Status** — whether the operation succeeded or failed
- **Links** — connections between spans from different traces, useful for async or batch processing

When you chain these across services, you get a full distributed trace — a visual timeline of what happened, where, and how long each step took.

## Structured Logs with ECS

For logs, we use the **Elastic Common Schema (ECS)** format. The benefits are simple: no manual parsing, a human-readable JSON structure, and easy integration with the ELK stack. When your logs follow a consistent schema, correlating them with traces and metrics becomes trivial.

## Metrics and Alerts

Metrics give you the quantitative foundation. But metrics without alerts are just dashboards nobody checks at 3 AM.

The important part isn't just having alerts — it's managing them properly. This is where the "as Code" part of the talk title comes in.

## Terraform-Managed Alerts

This was the section that got the most engagement. Instead of configuring alerts through a UI, we define them in **Terraform**:

- **Infrastructure as Code** — alerts are managed just like the rest of your infrastructure
- **Environment consistency** — the same alert configurations deploy automatically across dev, staging, and prod
- **Version control and traceability** — every change is tracked in Git with full review and rollback capability
- **Reduced human error** — automation eliminates manual misconfigurations
- **Scalability** — manage alerts across multiple services from one place

When someone asks "who changed this alert threshold and why?" — the answer is in the Git history, not lost in a UI audit log nobody checks.

## On-Call Process

The talk also covered how alerts connect to the on-call workflow. Having great observability is pointless if the alert fires and nobody responds effectively. The key is enriching notifications with enough context — service name, error details, direct links to traces — so the on-call engineer can understand the issue in seconds, not minutes.

## What's Next

I closed the talk with a roadmap slide:

1. Instrument all background jobs, queues, and cron tasks with OpenTelemetry for full trace coverage
2. Expand dashboards with business KPIs — conversion rate, order latency, failed payments
3. Automate alert routing and escalation for faster on-call response
4. Use AI-powered insights to predict incidents before they affect users
5. Integrate observability checks into CI/CD pipelines for automatic validation

## Key Takeaway

Observability isn't a tool you install — it's a practice you build. Start with the three pillars, standardize on OpenTelemetry, manage your alerts as code through Terraform, and connect everything to your on-call process. The goal is a system where you can **see it, trace it, and fix it** — before your users even notice.
