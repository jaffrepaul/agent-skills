---
name: sentry-otel-exporter-setup
description: Configure the OpenTelemetry Collector with Sentry Exporter for multi-project routing and automatic project creation. Use when setting up OTel with Sentry, auto-creating Sentry projects for new services, configuring collector pipelines for traces/logs, or routing telemetry from multiple services to separate Sentry projects.
---

# Sentry OTel Exporter Setup

Configure the OpenTelemetry Collector to send traces and logs to Sentry using the native Sentry Exporter.

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
    description: "Projects are created automatically based on `service.name` attribute — no manual setup needed."
```

**Default to "No"** in the generated config, but let the user decide.

## Routing Options

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

Services not in the mapping fall back to using `service.name` as project slug. Use `project_from_attribute` to route by a different attribute (e.g., `deployment.environment`).

## Configure Apps

Apps must set the `service.name` resource attribute (or configured routing attribute). This value becomes the Sentry project slug. Missing or empty values drop the data with a warning.
