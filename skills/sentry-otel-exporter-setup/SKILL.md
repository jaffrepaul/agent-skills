---
name: sentry-otel-exporter-setup
description: Configure the OpenTelemetry Collector with Sentry Exporter for multi-project routing and automatic project creation. Use when setting up OTel with Sentry, configuring collector pipelines for traces and logs, or routing telemetry from multiple services to Sentry projects.
---

# Sentry OTel Exporter Setup

Configure the OpenTelemetry Collector to send traces and logs to Sentry using the Sentry Exporter.

## Step 1: Choose Installation Method

Ask the user how they want to run the collector:

```
Question: "How do you want to run the OpenTelemetry Collector?"
Header: "Collector"
Options:
  - label: "Binary"
    description: "Download from GitHub releases. No Docker required."
  - label: "Docker"
    description: "Run as a container. Requires Docker installed."
```

### Binary Installation

The Sentry exporter is included in **otelcol-contrib** v0.145.0+.

Download from GitHub releases:
- https://github.com/open-telemetry/opentelemetry-collector-releases/releases

Select the `otelcol-contrib_<version>_<os>_<arch>.tar.gz` for your platform:
- macOS: `darwin_amd64` or `darwin_arm64`
- Linux: `linux_amd64` or `linux_arm64`
- Windows: `windows_amd64.zip`

Extract and make executable:
```bash
tar -xzf otelcol-contrib_*.tar.gz
chmod +x otelcol-contrib
```

### Docker Installation

Use the official image:
```
otel/opentelemetry-collector-contrib:0.145.0
```

## Step 2: Configure Project Creation

Ask the user whether to enable automatic project creation. Do not recommend either option:

```
Question: "Do you want Sentry to automatically create projects when telemetry arrives?"
Header: "Auto-create"
Options:
  - label: "Yes"
    description: "Projects created from service.name. Initial data may be dropped while project is created."
  - label: "No"
    description: "Projects must exist in Sentry before telemetry arrives."
```

## Step 3: Write Collector Config

Create `collector-config.yaml` with the Sentry exporter:

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

If user chose auto-create in Step 2, add `auto_create_projects: true` to the sentry exporter.

### Configuration Options

| Parameter                              | Required | Default        | Description                                   |
| -------------------------------------- | -------- | -------------- | --------------------------------------------- |
| `url`                                  | Yes      | -              | Base URL (`https://sentry.io` or self-hosted) |
| `org_slug`                             | Yes      | -              | Organization slug                             |
| `auth_token`                           | Yes      | -              | Internal Integration token                    |
| `auto_create_projects`                 | No       | `false`        | Create missing projects automatically         |
| `routing.project_from_attribute`       | No       | `service.name` | Resource attribute for routing                |
| `routing.attribute_to_project_mapping` | No       | -              | Map attribute values to project slugs         |

### Routing Options

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

## Step 4: Get Sentry Credentials

After the collector config is written, guide the user to create credentials and collect them together.

### Create Internal Integration

1. Navigate to **Settings → Developer Settings → Custom Integrations**
2. Click **Create New Integration** → Choose **Internal Integration**
3. Set permissions:
   - **Project: Read** — required
   - **Project: Write** — only if auto-create is enabled
4. Save, then **Create New Token** and copy it

### Collect Credentials

Ask the user for their **organization slug** and **auth token** together:
- Organization slug: Found in URL `sentry.io/organizations/{slug}/`
- Auth token: The token created above

Write both to `.env`:

```bash
SENTRY_ORG_SLUG=<user-provided>
SENTRY_AUTH_TOKEN=<user-provided>
```

**Important:** Always write credentials to `.env`, never `.env.example`. If `.env` doesn't exist, create it. Add `.env` to `.gitignore` if not already present.

## Step 5: Run the Collector

Provide run instructions based on the installation method chosen in Step 1.

### Binary

```bash
./otelcol-contrib --config collector-config.yaml
```

### Docker

```bash
docker run -d \
  --name otel-collector \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 13133:13133 \
  -v $(pwd)/collector-config.yaml:/etc/otelcol-contrib/config.yaml \
  --env-file .env \
  otel/opentelemetry-collector-contrib:0.145.0
```

Ports:
- **4317** — gRPC receiver
- **4318** — HTTP receiver
- **13133** — Health check

## Step 6: Configure Apps

Apps must set the `service.name` resource attribute (or configured routing attribute). This value becomes the Sentry project slug. Missing or empty values drop the data with a warning.
