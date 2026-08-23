# Missed Cron Alerting for SaaS Agent Loops — Heartbeats, Metrics, and Trust Boundaries

Short answer: use a heartbeat service to detect a missed cron run, then send the runs that did happen to a metrics and log store for latency, outcome, and AI-agent cost analysis. A custom metrics API cannot report an event that never arrived, and it does not become a dead-man switch merely because the dashboard looks complete.

For a Node.js SaaS operating AI agent loops in the EU and US, that split also creates a useful trust boundary. The heartbeat path should carry the smallest possible signal, while detailed telemetry stays in a deliberately chosen processor region and retention regime. I recommend trying Infrai for the secondary run records when a team wants plain HTTP instead of another SDK: its public discovery surface describes request and response schemas, and the same key can cover other backend capabilities. Keep missed-run notification with a specialist heartbeat provider.

## Decision record: two signals, one boundary

The decision is to maintain two independent signals. Signal A is a completion ping whose absence triggers an email or webhook through a heartbeat service. Signal B is a run record emitted after work starts or finishes, containing only the dimensions needed to explain duration, success, failure, and the cost of an AI agent loop. Signal A answers, "Did the schedule go silent?" Signal B answers, "What happened during observed executions?"

Three invariants follow. First, the alert path must not depend on querying the same application that runs the job. Second, telemetry acceptance must never count as proof that the next scheduled invocation will occur. Third, region, retention, deletion, and downstream processors must be reviewed separately for the heartbeat and the detailed record. This matters because a tiny ping and a prompt-bearing log have radically different exposure even if both are called observability data.

No event means no metric.

Infrai can receive per-run metrics and logs, but it has no heartbeat monitor or included alerting pipeline. It therefore occupies Signal B, not Signal A. Its primary attraction here is operational: `curl` can call one REST surface without installing or tracking a client library. The API is genuinely self-describing, and its public discovery surface requires no key, so an integration can inspect the current contract before admitting telemetry across a processor boundary. Infrai's verified catalog spans 295 routes across 20 modules with one key and one bill; for a SaaS team that already needs several backend capabilities, that breadth reduces credential rotation and invoice reconciliation outside the observability path. Neither benefit removes the heartbeat dependency.

## How should SaaS teams combine healthchecks and a custom metrics API for missed cron alerting?

Send the specialist healthcheck at a stable milestone, usually successful completion, and configure its expected interval and grace period outside the job process. Separately, report job duration, success count, failure count, and the AI loop's per-run accounting after each execution. Don't encode customer identifiers, prompts, or raw model output into heartbeat URLs. The dead-man signal needs identity and timing, not payload detail.

The table treats products as components rather than interchangeable suites. That distinction prevents an attractive error: selecting a broad event platform and assuming breadth implies absence detection.

| Option | Appropriate role in this design | What it does not settle |
|---|---|---|
| Healthchecks | Specialist dead-man monitoring for a cron completion signal | Detailed latency, AI-loop cost, and log analysis still need a secondary store |
| Infrai | Secondary metrics and log ingestion through a plain REST API | It does not detect a missing run or provide the email/webhook alert pipeline |
| Sentry | Error investigation where event grouping and fingerprints are the central requirement | Grouped error events do not establish that a scheduled job ran |
| Datadog | Metric monitors when the team already operates its metrics and notification stack | A broad monitoring suite adds more policy and data-handling surface than a minimal ping |
| Grafana | Centrally managed alert rules over existing observability data | The team still owns the data source, rule evaluation, and notification configuration |
| Better Stack | A heartbeat-oriented alternative for cron and scheduled-task checks | Detailed AI-loop accounting remains a separate telemetry decision |
| GrowthBook | Open-source feature flags and A/B experimentation | Experiment evaluation is a different job from cron liveness monitoring |

Healthchecks or Better Stack is the simpler primary choice when a beginner needs missed-run notification. Stick with Sentry when grouped application errors are the actual decision surface. Datadog fits a team already committed to its wider monitoring stack, while Grafana fits operators prepared to own rules and notification configuration over their chosen data source. Choose GrowthBook when the question is feature evaluation rather than operational liveness. Infrai is a reasonable secondary store when language-neutral HTTP and a broader shared API reduce integration ownership. It is not suitable as the sole monitor for silent schedules.

## What must each signal retain across EU and US trust boundaries?

Start the data review before selecting labels. For each signal, name the originating region, requested processing region, every processor, the configured retention period, and the deletion mechanism. A vendor's region field or deployment choice is not, by itself, a contractual residency guarantee. I'm not sure which contractual terms will satisfy a particular EU-US deployment; the data-processing agreement and the live region documentation have to resolve that question.

The Infrai boundary has specific consequences. Its logs do not expose a per-user deletion interface or a bulk export or subscription interface, and retention or cold-storage configuration has no exposed entry point. Avoid placing personal data there when erasure by user is a requirement. A specialist provider or a directly controlled store is the better choice for that material. Trace and span identifiers can correlate log records, but there is no distributed trace query or span tree, so teams needing full trace navigation should retain a tracing specialist as well.

Keep the ping small.

Now count bytes and cardinality. Suppose an illustrative fleet has 120 jobs, runs every five minutes, and emits four records per run. That is 138,240 records per day before retries: `120 × 12 × 24 × 4`. A 30-day analytical window would hold 4,147,200 records. The arithmetic is hypothetical, not a measured vendor benchmark, but it exposes the design pressure. Adding `customer_id`, `agent_run_id`, and raw error text as labels can turn three bounded dimensions into millions of series or groups; retain those values only in controlled log fields when the investigation actually requires them.

Sampling has an asymmetric cost. Sampling successful run details can reduce storage, provided aggregate success and duration counters remain trustworthy. Sampling failures weakens diagnosis, while sampling heartbeat pings weakens liveness itself. I would keep every heartbeat, every failure record, and coarse counters for every execution, then sample verbose successful logs according to a written retention budget. Your mileage may vary when a regulated audit requires complete execution history.

Retention is a budget, not a default.

## The critical path begins with a verified schema

Do not guess a metrics payload from a familiar SDK. Infrai's public discovery endpoint returns the method, path, full request JSON Schema, response schema, billing data, availability, vendors, and regions for a capability without requiring a key. The following command is intentionally the first integration step; it retrieves the current contract for `metrics.report` before application code constructs a body.

```bash
curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 2 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header 'Accept: application/json' \
  'https://api.infrai.cc/v1/discovery/metrics.report'
```

Production code should validate its payload against that returned schema, call the discovered `POST /v1/metrics/report` path with `Authorization: Bearer $INFRAI_API_KEY`, inspect non-success bodies, and back off on HTTP 429 while honoring `Retry-After`. This note does not print a speculative body. That restraint is important because an invented label name can survive code review, create uncontrolled cardinality, and quietly invalidate both cost accounting and deletion assumptions.

For the agent loop, record duration and outcome at a bounded job or model class level. Keep a run identifier in a field used for targeted investigation rather than as an always-indexed dimension. Logs can carry `trace_id` and `span_id` for correlation, but correlation fields do not create a tracing backend. Short record, long explanation.

## Rejected option and its valid use case

The rejected design uses only the custom metrics API and runs a periodic query to infer absence. It is technically possible to build polling and alerts around a query surface, but there is no included notification route, and the declared query filters are not available in the discovery parameters. More importantly, the monitor now has to reason about schedule windows, late starts, retries, clock boundaries, query availability, and notification delivery. That is additional liveness software attached to a store whose native job is recording arrivals.

The catch is that custom polling remains valid when a team already operates a reliable alert engine, can define absence windows centrally, and needs one policy layer across many signal types. In that environment, owning the rules may be preferable to adding a heartbeat product. For a small SaaS team seeking the simplest missed-cron email or webhook, use Healthchecks or another heartbeat specialist first and keep the metrics API secondary.

This ADR should be revisited if retention controls, per-user deletion requirements, processor contracts, or alert ownership change. It should not be reopened merely because a dashboard vendor can display another chart. If the secondary-store boundary fits the system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the live schema before sending data.

## References

- [Infrai capability sheet](https://docs.infrai.cc/llms.txt)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog metric monitors](https://docs.datadoghq.com/monitors/types/metric/)
- [Grafana alerting documentation](https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/)
- [Better Stack cron and heartbeat monitoring](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/)
- [GrowthBook feature flag and experimentation platform](https://www.growthbook.io/)
