# Cloud Metrics — SigNoz

Ship Temporal Cloud OpenMetrics to SigNoz through an OpenTelemetry Collector.
Use this file when a user asks to wire SigNoz up to Temporal Cloud, land the
`temporal_cloud_v1_*` metrics in the SigNoz **Metrics Explorer**, and import the
pre-built dashboard.

SigNoz is not a serverless integration — the OpenTelemetry Collector is your
responsibility. Temporal exposes the OpenMetrics endpoint at
`metrics.temporal.io` ,
and the Collector scrapes it with Bearer-token auth
 and forwards
metrics to your SigNoz ingestion endpoint over OTLP
.

---

## Prerequisites

Complete the OpenMetrics [Quickstart](https://docs.temporal.io/cloud/metrics/openmetrics#quickstart)
first. It requires the **Account Owner** or **Global Admin** role — a Namespace
Admin cannot grant the Metrics Read-Only role.

1. In the Temporal Cloud UI, go to **Settings → Service Accounts → Create Service
   Account** and assign the **Metrics Read-Only** account-level role.
2. Open the Service Account and create an API key. The key is shown only once —
   store it somewhere secure.
3. Verify the endpoint is reachable before configuring the Collector:

   ```shell
   curl -H "Authorization: Bearer <API_KEY>" https://metrics.temporal.io/v1/metrics
   ```

   You should see OpenMetrics-formatted output beginning with `# TYPE
   temporal_cloud_v1_...`.

`metrics.temporal.io` is for scrapers, not browsers — every request needs the
`Authorization: Bearer <API key>` header. Opening the URL in a browser returns
`Jwt is missing`.

You also need a SigNoz account (cloud or self-hosted), a SigNoz ingestion key,
and the ingestion endpoint for your SigNoz region. Those are owned by SigNoz —
see [signoz.io/docs/integrations/temporal-cloud-metrics/](https://signoz.io/docs/integrations/temporal-cloud-metrics/).

---

## Procedure

The SigNoz page ships the full Collector YAML — including a Kubernetes
Secret-based variant. Copy it verbatim from SigNoz's docs rather than hand-rolling
the config.

### Step 1 — Configure the OpenTelemetry Collector

Configure the Collector with:

- a **Prometheus receiver** against `metrics.temporal.io` using **Bearer** auth
  with your Temporal Cloud API key, and
- an **OTLP exporter** that sends to your SigNoz ingestion endpoint (using your
  SigNoz ingestion key and region).

Copy the full template from the [SigNoz integration page](https://signoz.io/docs/integrations/temporal-cloud-metrics/).
The Kubernetes Secret-based variant is on the same page.

Temporal's own docs publish the generic Prometheus-receiver shape the SigNoz
config is built on top of — use it as a shape check when reading the SigNoz
template:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
      - job_name: 'temporal-cloud'
        scrape_interval: 30s
        scrape_timeout: 30s
        honor_timestamps: true
        scheme: https
        authorization:
          type: Bearer
          credentials_file: <API_KEY_FILE>
        static_configs:
          - targets: ['metrics.temporal.io']
        metrics_path: '/v1/metrics'
```

The `scrape_interval` should be 30s — intervals longer than 60s may skip
datapoints because metrics update once per minute, and the account rate limit is
180 requests per account per hour.

`honor_timestamps: true` is required — the endpoint returns metrics from the
current timestamp minus a fixed 3-minute offset, and honoring the timestamp is
how Prometheus-compatible receivers preserve that offset.

### Step 2 — Deploy the Collector and confirm ingestion

Deploy the Collector as a VM/binary or in Kubernetes, then open the SigNoz
**Metrics Explorer** and confirm metrics with the `temporal_cloud_v1_` prefix are
arriving. Data typically appears within a few minutes.

If the Metrics Explorer is empty after several minutes, walk back down the chain:

1. Re-run the `curl` check from Prerequisites step 3 — this confirms the API key
   and endpoint. If it returns `Jwt is missing` or 401, the key or the Service
   Account role is wrong.
2. Check Collector logs for scrape errors against `metrics.temporal.io` and
   export errors against the SigNoz OTLP endpoint.
3. If `curl` returns metrics but SigNoz shows none, the failure is on the OTLP
   side (endpoint URL, ingestion key, or region). Follow the [SigNoz integration
   page](https://signoz.io/docs/integrations/temporal-cloud-metrics/) for
   SigNoz-side debugging.

### Step 3 — Import the pre-built dashboard

Download the pre-built dashboard JSON from the [SigNoz integration page](https://signoz.io/docs/integrations/temporal-cloud-metrics/)
and import it into SigNoz to visualize Temporal Cloud metrics.

---

## Cardinality and rate-limit awareness

- The endpoint truncates responses over **30,000 data points per scrape** and
  responds with `X-Completeness: limited`. Reduce cardinality with the
  `namespaces` and `metrics` query parameters if you see this.
- The account is capped at **180 requests per hour** across all scrapers. Two
  Collectors scraping every 30s already sits at that ceiling.
- Scrape **timeout** should be 10 seconds for large responses.

For advanced label management (dropping or bucketing `temporal_task_queue` /
`temporal_workflow_type` at the Collector), see the OpenTelemetry Collector
`filter` example in the API reference.

---

## Common mistakes

- **Hitting `https://metrics.temporal.io/metrics` instead of `/v1/metrics`.** The
  scrape path is `/v1/metrics`.
- **Trying to open the endpoint in a browser.** There is no browser UI; the
  endpoint returns `Jwt is missing` without the Bearer header.
- **Granting Metrics Read-Only from a Namespace Admin account.** Only Account
  Owner or Global Admin can grant that role.
- **Confusing Cloud OpenMetrics with SDK metrics.** Cloud OpenMetrics reports
  Task Queue backlog, workflow lifecycle, and account throttling; worker-side
  saturation, Schedule-To-Start latency, slot availability, and sticky-cache
  behavior come from [SDK metrics](https://docs.temporal.io/cloud/metrics/sdk-metrics-setup)
  configured on your Workers. Ingesting only Cloud metrics leaves worker
  bottlenecks invisible.
- **Setting `scrape_interval` to `60s` or longer.** Metrics update once per
  minute; longer intervals skip datapoints. Use 30s.
- **Omitting `honor_timestamps: true`.** The endpoint's 3-minute offset is
  carried in the response timestamp; dropping it produces gaps and duplicates.

---

## Related references

- [cloud-iam.md](cloud-iam.md) — Service Account and API key lifecycle
  (`tcld apikey`, `tcld service-account`).
- [rate-limits.md](../triage/rate-limits.md) — interpreting the throttle metrics
  on Cloud v1 (`temporal_cloud_v1_*`) once they are landing in SigNoz.
- [performance-bottlenecks.md](../triage/performance-bottlenecks.md) — SDK-metric
  playbook for worker-side latency and saturation, the counterpart to the Cloud
  OpenMetrics view.

---

## External sources

- [Temporal Cloud OpenMetrics — Metrics integrations (SigNoz section)](https://docs.temporal.io/cloud/metrics/openmetrics/metrics-integrations#signoz)
- [Temporal Cloud OpenMetrics — API reference](https://docs.temporal.io/cloud/metrics/openmetrics/api-reference)
- [Temporal Cloud OpenMetrics — Quickstart](https://docs.temporal.io/cloud/metrics/openmetrics#quickstart)
- [SigNoz integration page](https://signoz.io/docs/integrations/temporal-cloud-metrics/) — Collector YAML (VM and Kubernetes Secret variants), ingestion endpoints by region, dashboard JSON.
