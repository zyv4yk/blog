---
title: "Observability as Code: My Backend Meetup Talk, Now in Writing"
date: 2024-11-11
description: "A written adaptation of my Backend Meetup talk on observability in microservices — from the three pillars and OpenTelemetry to Terraform-managed alerts and on-call workflows."
tags: ["Observability", "OpenTelemetry", "Alerting", "Terraform", "Platform Engineering", "Meetup"]
---

A few weeks ago I gave a talk called **"Observability as Code: See It, Trace It, Fix It"** at our Backend Meetup at BetterMe. This post is a written version for anyone who couldn't make it — or just prefers reading over slides.

> 📺 Prefer to watch? Full talk recording on YouTube: [Observability as Code: See It, Trace It, Fix It](https://youtu.be/eCPFoH8Z4Ak?si=Q5KL6fPlsz0_fvi4)

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

A minimal Go example — wrap a request handler in a span, attach attributes, record errors:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
)

func handleCheckout(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("checkout").Start(r.Context(), "process-checkout")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.id", userID),
        attribute.Float64("order.total", total),
    )

    if err := charge(ctx, total); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "payment failed")
    }
}
```

The same pattern works in PHP via `open-telemetry/opentelemetry-php` — start a span, set attributes, end it.

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

A typical line looks like this:

```json
{
  "@timestamp": "2024-11-11T14:23:01.234Z",
  "log.level": "error",
  "service.name": "checkout-api",
  "trace.id": "1a2b3c4d5e6f7890abcdef1234567890",
  "span.id": "0a1b2c3d4e5f6789",
  "http.request.method": "POST",
  "http.response.status_code": 502,
  "error.message": "payment provider timeout"
}
```

Drop that into Kibana, click `trace.id`, and you're looking at the full distributed trace for that exact request — without any manual stitching.

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

A skeleton of what an alert resource looks like in Terraform:

```hcl
resource "monitoring_alert" "high_error_rate" {
  name     = "high_error_rate_${var.service}"
  interval = "1m"

  condition {
    metric    = "error.rate"
    threshold = 0.05      # 5% error rate
    window    = "5m"
  }

  destination = var.oncall_destination

  tags = {
    service     = var.service
    owner       = var.team
    runbook_url = "https://docs/runbooks/${var.service}-errors"
  }
}
```

The `runbook_url` tag is the small detail that pays off at 3 AM — the on-call engineer gets a link to "what to do" right inside the alert.

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

## Related reading

- [Building a Centralized CI/CD Platform for Microservices](/posts/ci-cd-platform-at-scale/) — where the "as Code" pattern came from
- [Supply Chain Security in Practice](/posts/supply-chain-security/) — same Infrastructure-as-Code principle, applied to SLSA and OIDC
