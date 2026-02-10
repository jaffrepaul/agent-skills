---
name: sentry-otel-exporter-setup
description: Configure the OpenTelemetry Collector with Sentry Exporter for multi-project routing and automatic project creation. Use when setting up OTel with Sentry, auto-creating Sentry projects for new services, configuring collector pipelines for traces/logs, or routing telemetry from multiple services to separate Sentry projects.
---

# Sentry OTel Exporter Setup

Configure the OpenTelemetry Collector to send traces and logs to Sentry using the native Sentry Exporter.

**Key benefit:** Automatically create Sentry projects when new services come online — no manual project setup required.

## Invoke This Skill When

- User wants Sentry to automatically create projects for new services
- User asks to "set up OTel with Sentry" or "configure OpenTelemetry for Sentry"
- User wants to route telemetry from multiple services to different Sentry projects
- User asks about `otelcol-contrib`, collector config, or Sentry exporter
- User wants to replace DSN-based routing with org-level authentication

## When to Use `sentry` vs `otlphttp`

| Scenario                                    | Exporter                             |
| ------------------------------------------- | ------------------------------------ |
| Single project, all services share one DSN  | `otlphttp`                           |
| Multiple projects, per-service routing      | `sentry`                             |
| Dynamic environments with auto-provisioning | `sentry` with `auto_create_projects` |

## Prerequisites

- **otelcol-contrib** — The Sentry exporter is included in [otelcol-contrib](https://github.com/open-telemetry/opentelemetry-collector-releases/tree/main/distributions/otelcol-contrib)
- Sentry organization with admin access to create Custom Integrations

## Phase 1: Create Sentry Auth Token

Guide user to create Internal Integration:

1. Navigate to **Settings → Developer Settings → Custom Integrations**
2. Click **Create New Integration** → Choose **Internal Integration**
3. Set permissions:
   - **Project: Read** — required
   - **Project: Write** — required for `auto_create_projects`
4. Save, then **Create New Token** and copy it

### Detect Organization Slug

Check for existing Sentry configuration to find org slugs:

```bash
# Check environment variables
env | grep -i sentry

# Check common config files
grep -r "org_slug\|organization" . --include="*.yaml" --include="*.yml" --include="*.env" 2>/dev/null | head -20
```

**If multiple org slugs are found:** Use `AskUserQuestion` to prompt the user to select the correct organization:

```
Question: "Which Sentry organization should receive telemetry?"
Header: "Org"
Options: [list each discovered org slug with description of where it was found]
```

**If no org slug is found:** Ask user to provide it manually. They can find it at **Settings → General Settings** or from the URL `https://sentry.io/organizations/{org-slug}/`

## Phase 2: Configure Collector

### Configuration Options

| Parameter                              | Required | Default        | Description                                   |
| -------------------------------------- | -------- | -------------- | --------------------------------------------- |
| `url`                                  | Yes      | -              | Base URL (`https://sentry.io` or self-hosted) |
| `org_slug`                             | Yes      | -              | Organization slug                             |
| `auth_token`                           | Yes      | -              | Internal Integration token                    |
| `auto_create_projects`                 | No       | `false`        | Create missing projects automatically         |
| `routing.project_from_attribute`       | No       | `service.name` | Resource attribute for routing                |
| `routing.attribute_to_project_mapping` | No       | -              | Map attribute values to project slugs         |

### Basic Config

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  sentry:
    url: https://sentry.io
    org_slug: ${env:SENTRY_ORG_SLUG}
    auth_token: ${env:SENTRY_AUTH_TOKEN}

processors:
  batch:

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [sentry]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [sentry]
```

### Environment Variables

```bash
export SENTRY_ORG_SLUG=YOUR_SLUG_HERE
export SENTRY_AUTH_TOKEN=YOUR_TOKEN_HERE
```

## Phase 3: Configure Project Creation

**Always ask the user** whether to enable automatic project creation using `AskUserQuestion`:

```
Question: "Do you want Sentry to automatically create projects when telemetry arrives?"
Header: "Auto-create"
Options:
  - label: "No"
    description: "You'll create projects manually in Sentry. May require manual routing config if you have multiple services."
  - label: "Yes"
    description: "Projects are created automatically based on service name — no manual setup needed."
```

**Default to "No"** in the generated config, but let the user decide.

## Routing Options

### Automatic Project Creation

Automatically creates Sentry projects when telemetry arrives — no need to manually set up projects beforehand:

```yaml
exporters:
  sentry:
    # ... required fields
    auto_create_projects: true
```

**Note:** Project creation is asynchronous. The first batch of data for a brand new project may be dropped while provisioning completes.

### Custom Project Mapping

Map service names to different project slugs:

```yaml
exporters:
  sentry:
    # ... required fields
    routing:
      attribute_to_project_mapping:
        orders-service: ecommerce-orders
        products-service: ecommerce-products
```

Services not in the mapping fall back to using `service.name` as project slug.

### Route by Different Attributes

Route by environment, team, or any resource attribute:

```yaml
exporters:
  sentry:
    # ... required fields
    routing:
      project_from_attribute: deployment.environment
      attribute_to_project_mapping:
        production: prod-monitoring
        staging: staging-monitoring
```

## Self-Hosted Sentry

Set `url` to your instance (e.g., `https://sentry.example.com`). For custom TLS, configure `http.tls.ca_file`.

## Using with Sentry SDKs

If also using a Sentry SDK, disable SDK tracing to avoid duplicates — let the collector handle traces while errors go directly to Sentry.

## Configure Apps

Apps must set the routing attribute (default: `service.name`). This becomes the Sentry project slug.

### Environment Variables (Recommended)

Works with any language that has an OpenTelemetry SDK:

```bash
OTEL_SERVICE_NAME=api-gateway
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

For SDK-specific configuration, see [OpenTelemetry docs](https://opentelemetry.io/docs/languages/).

## Project Slug Requirements

Slugs must be lowercase letters, numbers, and hyphens only (no underscores, max 50 chars). If the routing attribute is missing or empty, data is dropped.

## Verification

1. Start collector and services
2. Send test requests
3. Check Sentry for projects matching service names
4. Navigate to **Explore → Traces** to see distributed traces

## Troubleshooting & Limitations

| Issue | Solution |
| ----- | -------- |
| 403 errors | Verify token has Project:Read and Project:Write |
| Projects not created | Use lowercase letters, numbers, hyphens only |
| First batch dropped | Pre-create projects or send warmup requests |
| Deleted projects cause 403 | Restart collector to evict cache |
| Single org per exporter | Deploy multiple exporters for multi-org |
| No metrics support | Use separate exporter for metrics |
| Max 1000 projects/queue | Deploy multiple exporters or pre-create projects |

Check logs: `docker logs otelcol-contrib 2>&1 | grep -i sentry`

Rate limits are automatically respected and retried.
