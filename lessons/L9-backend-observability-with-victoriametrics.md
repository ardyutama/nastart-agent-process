# Lesson 9 - Backend Observability with OpenTelemetry and VictoriaMetrics

> **What you'll learn:**
> - How to make the .NET API observable before adding more backend features
> - How structured logs, RED metrics, and traces answer different operational questions
> - How to use OpenTelemetry as the application contract and VictoriaMetrics products as the backend
> - How to protect user data and metric cardinality in a single-user v1 application
> - How to verify telemetry, dashboards, and alerts before relying on them

> **Prerequisites:** Complete L1 through L8 first. This lesson extends the accumulated `Nastart.Api` startup code and the Application-layer cascade work from L7. It does not change any v1 business rule, database schema, JWT claim, or cascade interface.

> **Out of scope for this lesson:**
> - Frontend telemetry and browser Web Vitals
> - Python FastAPI instrumentation; it joins the same collector in a later phase
> - Choosing a production notification channel or setting an SLO without traffic data
> - Multi-tenant VictoriaMetrics configuration, tenant labels, roles, outlets, or user identifiers

---

## 1. Define "Working" Before Adding Telemetry

Do not start by adding every available metric. Start with questions an operator must be able to answer without reading source code.

| On-call question | Primary signal | Supporting signal | Why it matters |
|---|---|---|---|
| Is the API serving requests successfully and quickly enough? | HTTP rate, 5xx ratio, p95/p99 latency | Trace for a slow or failed request | This is the user-visible availability signal. |
| Did a price commit complete its cost cascade, or leave recipe costs stale? | Bounded business counters and a cascade-duration histogram | `IngredientPriceCommitted` / `CostCascadeFailed` logs and a cascade span | C-5 requires a failed cascade to be visible without rolling back price history. |
| Which dependency caused a slow or failed request? | Dependency error rate and duration | ASP.NET Core, `HttpClient`, and EF Core spans | Metrics identify the trend; a trace identifies the failing hop. |
| Is the telemetry path itself healthy? | `vmagent`, `vmalert`, and storage component metrics | Collector logs | A healthy API with a blind telemetry pipeline is still an operational risk. |

The v1 product has no approved availability or latency SLO yet. Therefore this lesson uses alert-rule environment variables instead of inventing thresholds. Set them only after agreeing the target and observing staging or production baseline traffic.

### Signal responsibilities

| Signal | Use it for | Do not use it for |
|---|---|---|
| Logs | The exact handled event, bounded fields, error type, trace ID | Aggregates, user IDs, request bodies, auth headers |
| Metrics | Rate, error ratio, histogram percentiles, resource health | Ingredient IDs, recipe IDs, emails, raw URLs, error messages |
| Traces | Request/dependency timing and internal cost-cascade steps | Storing request or response bodies, JWTs, passwords, prices, or PII |

---

## 2. The Backend-First Topology

The application speaks OpenTelemetry Protocol (OTLP). VictoriaMetrics products own storage, query, buffering, and alert evaluation. This keeps business code independent of a monitoring vendor and gives the future Python service the same integration point.

```text
                         OTLP/HTTP
+------------------+  ----------------> +---------------------+
| Nastart.Api      |                    | OpenTelemetry       |
| - ILogger logs   |                    | Collector            |
| - Meter metrics  |                    | - batch              |
| - Activity traces|                    | - memory limit       |
+------------------+                    +----------+----------+
                                                  | | |
                  metrics OTLP                   | | | OTLP/HTTP
                                                  | | |
                                                  | | +------> VictoriaTraces
                                                  | +--------> VictoriaLogs
                                                  v
                                             vmagent
                                                  |
                                             remote write
                                                  v
                                         VictoriaMetrics single
                                                  |
                                             vmui / queries
                                                  |
                                               vmalert
                                                  |
                                    approved Alertmanager route
```

| Component | Responsibility in v1 |
|---|---|
| `Nastart.Api` | Emits only vendor-neutral OTel signals. It does not know VictoriaMetrics URLs. |
| OpenTelemetry Collector | The single telemetry egress point. It batches, limits memory, applies later sampling/redaction policies, and routes signals. |
| `vmagent` | Receives OTLP metrics, constrains promoted metric labels, buffers remote writes, and forwards them to VictoriaMetrics. It also scrapes VictoriaMetrics component health metrics. |
| VictoriaMetrics single-node | Metrics storage, MetricsQL/PromQL querying, and `vmui` for the initial small deployment. |
| VictoriaLogs | Structured log storage and querying. |
| VictoriaTraces | Trace storage and querying. |
| `vmalert` | Evaluates metric symptom alerts and persists alert state. |
| Alertmanager | Required only to deliver production alerts. The notification channel is an operational decision, not a product Telegram feature. |

### Why this shape

- **OpenTelemetry is the application contract.** The .NET API uses `ILogger`, `Meter`, and `ActivitySource`, which are all native .NET abstractions collected by OpenTelemetry.
- **The collector prevents per-service vendor configuration.** When the Python service is added, it sends OTLP to the same internal collector rather than learning three VictoriaMetrics ingestion URLs.
- **`vmagent` is the metric safety boundary.** It accepts OTLP at `/opentelemetry/v1/metrics`, buffers remote writes, and can restrict resource attributes before they become labels.
- **Single-node storage fits v1.** The application is a single-user product, not a multi-tenant SaaS. Do not add `vm_account_id`, `outletId`, role, company, or user labels to telemetry.
- **The visualization layer is deliberately deferred.** `vmui` and the VictoriaLogs/VictoriaTraces UIs are sufficient for the first verified deployment. Adopt Grafana only if a concrete dashboard workflow requires it.

---

## 3. Telemetry Data Contract

Telemetry is an external data store. Treat fields and labels as public operational contracts, not a dumping ground for application objects.

### Allowed resource attributes

Every signal may carry only these application-level resource attributes by default:

| Attribute | Example | Reason |
|---|---|---|
| `service.name` | `nastart-api` | Stable service identity |
| `service.version` | Build or release version | Deploy correlation |
| `deployment.environment.name` | `development`, `staging`, `production` | Environment separation |

`service.instance.id`, host, and process information can remain on logs and traces, but must not be promoted as metric labels. A restart-generated instance ID creates unnecessary metric series churn.

### Forbidden fields and labels

Never emit these in a log field, metric label, trace attribute, resource attribute, alert label, or collector header:

- JWTs, passwords, API keys, Telegram link codes, authorization headers, connection strings, or cookies
- Email addresses, names, Telegram usernames, request/response bodies, SQL parameter values, or full query strings
- `userId`, `ingredientId`, `recipeId`, `versionGroupId`, raw URLs, request IDs, error messages, or exception messages as **metric labels**
- Ingredient prices, cost-per-portion values, margins, or uploaded receipt text
- `outletId`, `role`, `companyId`, tenant IDs, or any v2-only concept

Opaque IDs may be useful during a local investigation, but they are not needed for the baseline v1 telemetry contract. Use the W3C trace ID and the application's audited records to correlate an incident instead.

### Stable log events

Use source-generated `LoggerMessage` methods. Their method names become stable event names in OpenTelemetry log records, their templates stay structured, and they avoid string interpolation.

| Event name | Level | Allowlisted fields |
|---|---|---|
| `IngredientPriceCommitted` | `Information` | `PriceSource`, `CascadeOutcome`, `RecalculatedRecipeCount` |
| `CostCascadePartialFailure` | `Warning` | `FailedRecipeCount`, `ErrorType` |
| `CostCascadeFailed` | `Error` | `ErrorType` |
| `ExternalCallFailed` | `Warning` | `Dependency`, `StatusClass`, `Attempt` |

An exception's type is an acceptable error field. Do not add `exception.Message` to a custom field. Keep EF Core sensitive-data logging disabled in production so exception text and generated SQL cannot expose values.

### Metric dimensions

Use only small, fixed sets:

| Metric | Allowed dimensions |
|---|---|
| `nastart.ingredient.price_commits` | `source` (`Manual`, `InvoiceScan`), `outcome` (`success`, `partial`, `failed`) |
| `nastart.cost_cascade.duration` | `outcome` (`success`, `partial`, `failed`) |
| `nastart.cost_cascade.failures` | `error.category` from a finite application-owned set (`database`, `unexpected`) |
| HTTP/dependency metrics | Route template, method, status class for aggregation, dependency name |

Automatic HTTP instrumentation retains standard OpenTelemetry status-code attributes. Alert and dashboard queries must aggregate them into `2xx`, `4xx`, and `5xx` classes. Do not introduce raw status codes into custom business metrics.

### Correlation rule

The W3C trace ID is the canonical request correlation ID. ASP.NET Core and OpenTelemetry propagate `traceparent` to supported outbound HTTP calls. Echo the current trace ID in `X-Request-Id` for support requests; it may continue a valid incoming `traceparent`, so it is a diagnostic value only and never an authorization or business field. Do not accept a caller-provided `X-Request-Id` as an application field.

---

## 4. Dependency Approval and Test-First Sequence

`AGENTS.md` requires approval before adding NuGet packages. This lesson documents the proposed package set; add it to the target application only after that approval.

```bash
dotnet add src/Nastart.Api package OpenTelemetry.Extensions.Hosting
dotnet add src/Nastart.Api package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add src/Nastart.Api package OpenTelemetry.Instrumentation.AspNetCore
dotnet add src/Nastart.Api package OpenTelemetry.Instrumentation.Http
dotnet add src/Nastart.Api package OpenTelemetry.Instrumentation.Runtime
dotnet add src/Nastart.Api package OpenTelemetry.Instrumentation.EntityFrameworkCore
```

Do not pin an unverified package version in a lesson. At implementation time, select mutually compatible, currently supported package versions, record the decision, and restore before writing application code.

Before wiring exporters, write the smallest tests that prove the telemetry contract:

1. A request integration test expects a non-empty, 32-character hexadecimal `X-Request-Id` response header.
2. A unit test using a `MeterListener` confirms the price-commit instrument emits only `source` and `outcome` tags, never an ingredient or user ID.
3. A handler/service test for a partial cascade confirms the emitted outcome is `partial` and the structured log call carries only allowlisted fields.
4. An integration test using PostgreSQL Testcontainers covers any EF Core/Npgsql behavior; do not substitute EF Core InMemory for relational telemetry paths.

If the target application has no suitable telemetry-test harness, stop before adding a test-only exporter package and ask for approval, as required by `AGENTS.md`.

---

## 5. Emit Business Signals from the Application Layer

The Application layer owns meaningful business telemetry because it owns the cost-cascade behavior. It depends only on .NET's built-in diagnostics APIs; exporter setup remains in `Nastart.Api`.

Create `src/Nastart.Application/Common/Observability/NastartTelemetry.cs`:

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

namespace Nastart.Application.Common.Observability;

public static class NastartTelemetry
{
    public const string ActivitySourceName = "Nastart.Application";
    public const string MeterName = "Nastart.Application";

    public static readonly ActivitySource ActivitySource = new(ActivitySourceName);
    public static readonly Meter Meter = new(MeterName);

    public static readonly Counter<long> IngredientPriceCommits =
        Meter.CreateCounter<long>("nastart.ingredient.price_commits", unit: "{commit}");

    public static readonly Counter<long> CostCascadeFailures =
        Meter.CreateCounter<long>("nastart.cost_cascade.failures", unit: "{failure}");

    public static readonly Histogram<double> CostCascadeDuration =
        Meter.CreateHistogram<double>("nastart.cost_cascade.duration", unit: "s");
}
```

Create `src/Nastart.Application/Common/Observability/ObservabilityLog.cs`:

```csharp
using Microsoft.Extensions.Logging;

namespace Nastart.Application.Common.Observability;

public static partial class ObservabilityLog
{
    [LoggerMessage(
        EventId = 1001,
        EventName = "IngredientPriceCommitted",
        Level = LogLevel.Information,
        Message = "Ingredient price commit completed. Source={PriceSource} CascadeOutcome={CascadeOutcome} RecalculatedRecipeCount={RecalculatedRecipeCount}")]
    public static partial void IngredientPriceCommitted(
        this ILogger logger,
        string priceSource,
        string cascadeOutcome,
        int recalculatedRecipeCount);

    [LoggerMessage(
        EventId = 1002,
        EventName = "CostCascadePartialFailure",
        Level = LogLevel.Warning,
        Message = "Cost cascade completed with isolated failures. FailedRecipeCount={FailedRecipeCount} ErrorType={ErrorType}")]
    public static partial void CostCascadePartialFailure(
        this ILogger logger,
        int failedRecipeCount,
        string errorType);

    [LoggerMessage(
        EventId = 1003,
        EventName = "CostCascadeFailed",
        Level = LogLevel.Error,
        Message = "Cost cascade failed. ErrorType={ErrorType}")]
    public static partial void CostCascadeFailed(
        this ILogger logger,
        string errorType);
}
```

In the existing price-commit and `CostCascadeService` paths, add telemetry next to the existing successful, partial, and failed outcomes. Do not change the C-1 interface or the C-5 failure-isolation behavior to make instrumentation easier.

```csharp
using System.Diagnostics;
using Nastart.Application.Common.Observability;

using var activity = NastartTelemetry.ActivitySource.StartActivity("cost_cascade");
var stopwatch = Stopwatch.StartNew();

try
{
    // Run the existing C-2 calculation and C-5 failure isolation here.
    // Map its existing result to one of: success, partial, failed.
    const string outcome = "success";

    activity?.SetTag("cascade.outcome", outcome);
    NastartTelemetry.CostCascadeDuration.Record(
        stopwatch.Elapsed.TotalSeconds,
        new TagList { { "outcome", outcome } });
}
catch (Exception exception)
{
    const string outcome = "failed";
    var errorType = exception.GetType().Name;
    const string errorCategory = "unexpected";

    activity?.SetStatus(ActivityStatusCode.Error);
    activity?.SetTag("error.type", errorType);
    activity?.SetTag("cascade.outcome", outcome);

    NastartTelemetry.CostCascadeFailures.Add(
        1,
        new TagList { { "error.category", errorCategory } });
    NastartTelemetry.CostCascadeDuration.Record(
        stopwatch.Elapsed.TotalSeconds,
        new TagList { { "outcome", outcome } });

    logger.CostCascadeFailed(errorType);
    throw;
}
```

For an ingredient price commit, the only source value is the existing C-13 enum value: `Manual` or `InvoiceScan`. Record it as the bounded `source` tag. Map cascade failures to a fixed application-owned category such as `database` or `unexpected` before recording a metric; the exception type may remain a trace/log field, but never a metric label. Do not emit the price, ingredient ID, authenticated user ID, or receipt contents.

---

## 6. Configure OpenTelemetry in `Nastart.Api`

The API project composes instrumentation and exporters. Domain and Application remain free of OpenTelemetry NuGet dependencies except for built-in `System.Diagnostics` types used above.

Add the following imports and registration before `var app = builder.Build()` in `src/Nastart.Api/Program.cs`:

```csharp
using System.Reflection;
using Nastart.Application.Common.Observability;
using OpenTelemetry.Exporter;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

const string serviceName = "nastart-api";

var otlpEndpoint = builder.Configuration["Telemetry:OtlpEndpoint"]
    ?? throw new InvalidOperationException(
        "Telemetry:OtlpEndpoint is not configured. Set it to the internal OpenTelemetry Collector endpoint.");

var traceSampleRatio = builder.Configuration.GetValue<double?>("Telemetry:TraceSampleRatio")
    ?? (builder.Environment.IsDevelopment() ? 1d : 0.1d);

if (traceSampleRatio is < 0d or > 1d)
{
    throw new InvalidOperationException("Telemetry:TraceSampleRatio must be between 0 and 1.");
}

var serviceVersion = typeof(Program).Assembly
    .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?
    .InformationalVersion
    ?? "unknown";

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: serviceName, serviceVersion: serviceVersion)
        .AddAttributes(new Dictionary<string, object>
        {
            ["deployment.environment.name"] = builder.Environment.EnvironmentName,
        }))
    .WithLogging(logging =>
    {
        logging.IncludeFormattedMessage = false;
        logging.ParseStateValues = true;
    })
    .WithTracing(tracing => tracing
        .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(traceSampleRatio)))
          .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
          .AddEntityFrameworkCoreInstrumentation()
          .AddSource(NastartTelemetry.ActivitySourceName))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
          .AddMeter(NastartTelemetry.MeterName))
        .UseOtlpExporter(OtlpExportProtocol.HttpProtobuf, new Uri(otlpEndpoint));
```

      Use an OTLP collector base URL with a trailing slash, such as `http://otel-collector:4318/`, not a VictoriaMetrics signal-specific path. `UseOtlpExporter` appends the correct `/v1/logs`, `/v1/metrics`, and `/v1/traces` paths; the collector owns its three downstream destination paths.

      Do not globally enable `IncludeScopes`: arbitrary future scopes can carry unreviewed data. OpenTelemetry still correlates supported logs with the current activity. Leave `OTEL_DOTNET_EXPERIMENTAL_EFCORE_ENABLE_TRACE_DB_QUERY_PARAMETERS` unset or `false`, and do not add `DbCommand` enrichers that expose query values. The current EF Core instrumentation supports relational providers such as Npgsql/PostgreSQL and leaves parameter capture disabled by default.

Set non-secret configuration in environment-specific application settings or deployment environment variables:

```json
{
  "Telemetry": {
    "OtlpEndpoint": "http://otel-collector:4318/",
    "TraceSampleRatio": 1.0
  }
}
```

Use `1.0` while verifying development and staging telemetry. A production head-sampling ratio belongs to an approved capacity decision; `0.1` is only the guarded code default, not an SLO or cost promise.

### Request correlation middleware

Keep the L8 exception-handler ordering intact. Immediately after `app.UseExceptionHandler()`, echo the current server trace ID. ASP.NET Core creates the request activity before the pipeline executes, and OpenTelemetry automatically attaches it to logs and spans.

```csharp
using System.Diagnostics;

app.UseExceptionHandler();

app.Use(async (context, next) =>
{
    var traceId = Activity.Current?.TraceId.ToString();

    if (!string.IsNullOrWhiteSpace(traceId))
    {
        context.Response.Headers["X-Request-Id"] = traceId;
    }

    await next();
});

app.UseAuthentication();
app.UseAuthorization();
```

Do not create a second correlation scheme, parse a caller-supplied request-ID header, or include a JWT claim in a log scope. The response header gives a support request a stable ID while OTel preserves trace context across supported HTTP boundaries.

### Sampling note

The registration above uses head sampling. If a production requirement needs every error trace retained, configure the API to export all traces and add a tail-sampling policy in the collector. Do not combine a 10% SDK head sample with a collector policy and assume the collector can recover traces the SDK never exported.

An example tail policy for a small deployment is:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 1000
    expected_new_traces_per_sec: 10
    policies:
      - name: retain-errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: sample-successes
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

service:
  pipelines:
    traces:
      processors: [memory_limiter, tail_sampling, batch]
```

Use that policy only after sizing the collector's memory and trace volume. When it is enabled, set the API's `Telemetry:TraceSampleRatio` to `1.0` so error traces reach the tail sampler.

---

## 7. Route Signals Through the Collector

Create the following future target structure alongside the deployment compose file:

```text
infra/observability/
  otel-collector.yaml
  vmagent.yaml
  rules/
    nastart-api.yaml
  runbooks/
    api-error-rate.md
    api-latency.md
    cascade-failure.md
```

Create `infra/observability/otel-collector.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256
    spike_limit_mib: 64
  batch:
    send_batch_size: 1024
    timeout: 5s

exporters:
  otlphttp/vmagent:
    metrics_endpoint: http://vmagent:8429/opentelemetry/v1/metrics
  otlphttp/victorialogs:
    logs_endpoint: http://victoria-logs:9428/insert/opentelemetry/v1/logs
  otlphttp/victoriatraces:
    traces_endpoint: http://victoria-traces:10428/insert/opentelemetry/v1/traces

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/vmagent]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/victorialogs]
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/victoriatraces]
```

This is intentionally a small first configuration. Add a collector transform or filter processor only after testing it against the pinned collector version. Prevention in application code remains the primary protection against sensitive data reaching telemetry.

### Configure `vmagent` as the metric guardrail

`vmagent` receives OTLP metrics from the collector and remote-writes them to single-node VictoriaMetrics. Pass these flags in the pinned image's Docker Compose service definition:

```text
-remoteWrite.url=http://victoria-metrics:8428/api/v1/write
-remoteWrite.tmpDataPath=/vmagent-data
-remoteWrite.maxDiskUsagePerURL=<approved-buffer-budget>
-opentelemetry.usePrometheusNaming
-opentelemetry.promoteAllResourceAttributes=false
-opentelemetry.promoteResourceAttributes=service.name,deployment.environment.name,service.version
-opentelemetry.promoteScopeMetadata=false
```

The first three flags establish the delivery path and bounded disk queue. The last three are a cardinality and privacy boundary:

- `-opentelemetry.usePrometheusNaming` gives MetricsQL/PromQL-compatible metric and label names.
- `-opentelemetry.promoteAllResourceAttributes=false` prevents every process, host, and future application resource attribute from becoming a label.
- `-opentelemetry.promoteResourceAttributes=...` allows only the three resource dimensions defined in this lesson.
- `-opentelemetry.promoteScopeMetadata=false` prevents instrumentation scope metadata from becoming unreviewed metric labels.

Replace `<approved-buffer-budget>` with a disk budget chosen from measured metric throughput and the desired outage window. Do not leave the queue unlimited in production.

Create `infra/observability/vmagent.yaml` to scrape the health metrics of the telemetry components themselves:

```yaml
global:
  scrape_interval: 30s
  scrape_timeout: 10s

scrape_configs:
  - job_name: vmagent
    static_configs:
      - targets: ["vmagent:8429"]

  - job_name: victoria-metrics
    static_configs:
      - targets: ["victoria-metrics:8428"]

  - job_name: victoria-logs
    static_configs:
      - targets: ["victoria-logs:9428"]

  - job_name: victoria-traces
    static_configs:
      - targets: ["victoria-traces:10428"]

  - job_name: vmalert
    static_configs:
      - targets: ["vmalert:8880"]
```

Expose the collector's own metrics only through a version-pinned, explicitly configured collector telemetry endpoint. Do not rely on an image's historical default listener. Add that scrape target after confirming the collector image's current configuration schema.

### Network and secret boundary

- Keep OTLP receiver port `4318`, VictoriaMetrics component ports, and `vmagent` ingestion ports on the private Docker network. Do not publish them to the public internet.
- Terminate TLS and authenticate telemetry ingress before a public network boundary. Never solve certificate issues with an insecure-skip-verify flag in production.
- Store any remote-write, Alertmanager, or collector credentials in deployment secrets or mounted secret files, never in Compose files, rule files, source, or telemetry URLs.
- Pin every collector and VictoriaMetrics image to a tested release. Do not deploy `latest`.

---

## 8. Alerts and Runbooks with `vmalert`

`vmalert` evaluates symptom alerts against VictoriaMetrics. Configure all three URLs so its alert state survives restarts:

```text
-datasource.url=http://victoria-metrics:8428
-remoteWrite.url=http://victoria-metrics:8428
-remoteRead.url=http://victoria-metrics:8428
-rule=/etc/vmalert/*.yaml
```

For local validation only, add `-notifier.blackhole`. Before production, replace it with an approved Alertmanager route. Do not send operational pages through the customer-facing Telegram bot without an explicit product and security decision.

Create `infra/observability/rules/nastart-api.yaml`:

```yaml
groups:
  - name: nastart-api-symptoms
    interval: 1m
    rules:
      - alert: NastartApiErrorRateHigh
        expr: |
          (
            sum(rate(http_server_request_duration_seconds_count{
              service_name="nastart-api",
              deployment_environment_name="production",
              http_response_status_code=~"5.."
            }[5m]))
            /
            sum(rate(http_server_request_duration_seconds_count{
              service_name="nastart-api",
              deployment_environment_name="production"
            }[5m]))
          ) > %{NASTART_API_MAX_5XX_RATE}
        for: "%{NASTART_API_5XX_FOR}"
        labels:
          severity: page
          service: nastart-api
        annotations:
          summary: "Nastart API error ratio exceeds the approved budget"
          runbook: "https://<approved-ops-docs>/runbooks/nastart-api-error-rate"

      - alert: NastartApiP99LatencyHigh
        expr: |
          histogram_quantile(
            0.99,
            sum by (le, http_route) (
              rate(http_server_request_duration_seconds_bucket{
                service_name="nastart-api",
                deployment_environment_name="production"
              }[5m])
            )
          ) > %{NASTART_API_MAX_P99_SECONDS}
        for: "%{NASTART_API_P99_FOR}"
        labels:
          severity: ticket
          service: nastart-api
        annotations:
          summary: "Nastart API p99 latency exceeds the approved target"
          runbook: "https://<approved-ops-docs>/runbooks/nastart-api-latency"

      - alert: NastartCascadeFailuresDetected
        expr: |
          sum(rate(nastart_cost_cascade_failures_total{
            service_name="nastart-api",
            deployment_environment_name="production"
          }[10m])) > 0
        for: "%{NASTART_CASCADE_FAILURE_FOR}"
        labels:
          severity: ticket
          service: nastart-api
        annotations:
          summary: "One or more recipe cost cascades failed"
          runbook: "https://<approved-ops-docs>/runbooks/nastart-cascade-failure"

      - alert: NastartTelemetryPipelineBacklog
        expr: |
          vmagent_remotewrite_pending_data_bytes > %{NASTART_VM_AGENT_MAX_PENDING_BYTES}
        for: "%{NASTART_VM_AGENT_BACKLOG_FOR}"
        labels:
          severity: ticket
          service: observability
        annotations:
          summary: "Telemetry delivery is backlogged at vmagent"
          runbook: "https://<approved-ops-docs>/runbooks/nastart-telemetry-backlog"
```

The variable placeholders prevent a guessed alert threshold from reaching production. Define the approved values in secure deployment configuration, not in source control. Check the emitted metric names in VictoriaMetrics after the first real ingestion; current OpenTelemetry-to-Prometheus naming produces the `http_server_request_duration_seconds_*` and `nastart_*` forms used above.

Each runbook must contain at least:

1. What the alert means for users and data correctness.
2. The first metric, trace, or log query to run.
3. The owner/escalation path and the safe mitigation.

Do not put a dynamic value in a `vmalert` label. It creates a new alert identity on every evaluation and makes `for` behavior unreliable. Put dynamic diagnosis detail in annotations instead.

---

## 9. Verify the Telemetry, Not Just the Startup Code

Run this sequence in a disposable local or staging environment. Never force failures in production solely to test observability.

1. Start the pinned VictoriaMetrics stack, collector, and API with `Telemetry:TraceSampleRatio=1.0`.
2. Call `/health` and one protected endpoint. Confirm the response includes `X-Request-Id`.
3. Create a normal ingredient price commit. Verify the `IngredientPriceCommitted` log record contains its event name, source, cascade outcome, trace ID, and no price or user data.
4. Cause a controlled cascade failure in staging through an existing safe test seam. Confirm C-5 still records `CascadeErrorLog`, the price history is not rolled back, and the `CostCascadeFailed` log/metric/span appears.
5. Query VictoriaMetrics for the HTTP counter and the `nastart_cost_cascade_*` series. Confirm resource labels are limited to service name, environment, and version.
6. Find the same request in VictoriaLogs and VictoriaTraces using the returned trace ID. The request span, EF Core/HTTP dependency spans, and logs must share that trace ID.
7. Check `vmagent` metrics for remote-write failures, dropped samples, queue growth, and series-limit drops. A successful application request is not proof that telemetry was delivered.
8. Validate rules before loading them:

   ```bash
   vmalert -rule=infra/observability/rules/nastart-api.yaml -dryRun
   ```

9. In staging only, temporarily lower an approved alert threshold, confirm it reaches the correct receiver with a working runbook link, and restore the normal threshold.
10. Run the target application's focused telemetry tests, then its full test command, build, and lint commands.

### Completion checklist

- [ ] Logs are structured, use stable event names, and carry a trace ID.
- [ ] Logs, traces, and metrics contain no secrets, PII, request bodies, prices, or unbounded labels.
- [ ] ASP.NET Core, `HttpClient`, runtime, and EF Core signals are exported through OTLP.
- [ ] Price commits and cost cascades emit bounded business signals without changing C-1 through C-5 or C-13 behavior.
- [ ] `vmagent` limits promoted resource attributes and has a finite delivery queue budget.
- [ ] VictoriaMetrics, VictoriaLogs, and VictoriaTraces receive real test telemetry.
- [ ] `vmalert` has persisted state, syntax-checked rules, symptom-based alerts, and runbook links.
- [ ] An induced staging failure was diagnosed from telemetry alone.

---

## 10. Sources and Follow-On Work

This lesson is based on the following official documentation, verified on 2026-08-14:

- [.NET observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [OpenTelemetry .NET tracing for ASP.NET Core](https://opentelemetry.io/docs/languages/dotnet/traces/getting-started-aspnetcore/)
- [OpenTelemetry .NET logging for ASP.NET Core](https://opentelemetry.io/docs/languages/dotnet/logs/getting-started-aspnetcore/)
- [VictoriaMetrics OpenTelemetry metrics ingestion](https://docs.victoriametrics.com/victoriametrics/integrations/opentelemetry/)
- [vmagent OTLP ingestion, persistent queue, and cardinality limits](https://docs.victoriametrics.com/victoriametrics/vmagent/)
- [VictoriaLogs OpenTelemetry ingestion](https://docs.victoriametrics.com/victorialogs/data-ingestion/opentelemetry/)
- [VictoriaTraces OpenTelemetry ingestion](https://docs.victoriametrics.com/victoriatraces/data-ingestion/opentelemetry/)
- [vmalert configuration, state persistence, validation, and alert rules](https://docs.victoriametrics.com/victoriametrics/vmalert/)

After the backend slice is verified, instrument the Python service with the same resource-attribute contract and OTLP collector endpoint. Treat browser telemetry as a separate privacy and consent decision rather than automatically adding it to the frontend.