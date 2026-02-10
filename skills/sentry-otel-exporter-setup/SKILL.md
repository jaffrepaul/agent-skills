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
| `timeout`                              | No       | `30s`          | Exporter timeout                              |
| `http`                                 | No       | collector defaults | HTTP client settings (timeout, TLS, headers) |
| `sending_queue`                        | No       | enabled        | Queue settings from exporterhelper            |

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

For self-hosted installations:

```yaml
exporters:
  sentry:
    url: https://sentry.example.com
    org_slug: ${env:SENTRY_ORG_SLUG}
    auth_token: ${env:SENTRY_AUTH_TOKEN}
    http:
      tls:
        ca_file: /path/to/ca.crt # Your CA certificate
```

**Warning:** Avoid `insecure_skip_verify: true` in production — it disables TLS verification.

## Using with Sentry SDKs

If you also use a Sentry SDK (e.g., `sentry-python`, `sentry-go`, `@sentry/node`), the exporter does not maintain trace connectedness by itself. Configure the SDK to avoid duplicate trace export:

| SDK | Configuration |
| --- | ------------- |
| sentry-go | Use OTLP integration with `setup_otlp_traces_exporter=false` |
| sentry-python | Set `traces_sample_rate=0` or filter with `before_send_transaction` |
| @sentry/node | Disable SDK tracing; let OTel handle traces |

This ensures traces flow through the collector while errors still go directly to Sentry.

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

| Requirement                         | Example             |
| ----------------------------------- | ------------------- |
| Lowercase letters, numbers, hyphens | `api-gateway` ✅    |
| No underscores                      | `orders_service` ❌ |
| No uppercase                        | `OrdersService` ❌  |
| Max 50 characters                   | -                   |

**Important:** If routing attribute is missing or empty, data is dropped with a warning.

## Verification

1. Start collector and services
2. Send test requests
3. Check Sentry for projects matching service names
4. Navigate to **Explore → Traces** to see distributed traces

## Troubleshooting

| Issue                     | Cause                              | Solution                                                   |
| ------------------------- | ---------------------------------- | ---------------------------------------------------------- |
| 403 errors                | Missing permissions                | Verify token has Project:Read and Project:Write            |
| Projects not created      | Invalid slug format                | Use lowercase letters, numbers, hyphens only               |
| First batch dropped       | Async project creation             | Pre-create projects or send warmup requests                |
| Data missing after delete | Collector cache                    | Restart collector to evict cache                           |
| 403 then succeeds         | Cache eviction triggered           | Normal behavior; exporter auto-retries after evicting stale entry |
| Partial batch failures    | Multi-project routing              | Retries not possible; some projects may receive duplicates |

### Check Collector Logs

```bash
docker logs otelcol-contrib 2>&1 | grep -i sentry
```

## Rate Limiting

The exporter automatically respects Sentry rate limits and retries throttled requests.

## Limitations

| Limitation                                    | Workaround                                    |
| --------------------------------------------- | --------------------------------------------- |
| Missing routing attribute drops data          | Ensure `service.name` is set on all resources |
| First batch for new projects may drop         | Pre-create projects or send warmup requests   |
| Deleted projects cause 403 until cache evicts | Avoid deleting projects while collector runs  |
| Single org per exporter                       | Deploy multiple exporters for multi-org       |
| No metrics support                            | Use separate exporter for metrics             |
| Partial failures can't retry cleanly          | Some projects may receive duplicates on retry |
| Max 1000 projects cached                      | Deploy multiple exporters if exceeded         |
| Auto-create queue limited to 1000             | Pre-create projects for large deployments     |
