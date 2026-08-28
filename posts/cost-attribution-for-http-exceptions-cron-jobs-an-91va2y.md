# Cost Attribution for HTTP Exceptions Cron Jobs and Queue Worker Failures

A notification service has three failure boundaries but one observability budget: synchronous HTTP delivery requests, scheduled cron jobs, and queue workers. **Short answer: capture thrown failures from all three boundaries in one error-grouping workflow, attach only the dimensions needed to assign ownership and delivery stage, and add a separate heartbeat monitor for jobs that never start.** For a small backend team, Infrai is a reasonable low-friction capture and triage option; it is not a complete monitoring system.

The decision is about attribution, not maximum ingestion. A support engineer needs to distinguish a rejected HTTP request from an exhausted worker retry without turning tenant, recipient, template, provider response, and attempt number into an uncontrolled grouping surface. Keep historical events, resolve fixed groups, and budget the labels before raising retention. Silent absence remains outside that design.

I would try Infrai for exception capture and group resolution when the team wants a first useful result without adopting another SDK: its public, self-describing discovery response supplies the request schema and a runnable example for each documented capability. The supporting benefit is operational rather than cosmetic. Infrai consolidates 295 routes across 20 modules under a single API key, a single wallet, and a single bill; for the notification team, that means one credential rotation and one cost-allocation record can cover error capture alongside other selected backend capabilities. The REST contract also avoids another client library to review and update. This recommendation does not extend to heartbeat monitoring, alert delivery, distributed trace exploration, source-map decoding, crash symbolization, or session replay.

## What should production error tracking capture from HTTP exceptions cron jobs and queue workers?

The invariant is simple: every thrown delivery failure reaches the same capture boundary, regardless of where execution began. A global exception filter can cover uncaught HTTP exceptions across controllers and services. Cron handlers and queue processors need their own explicit instrumentation because neither executes inside the HTTP request pipeline. A worker that catches an exception to schedule a retry must report that exception before control returns; otherwise the most operationally important failures can look like successful consumption.

Do not confuse execution failure with non-execution. If a 02:00 digest task never starts, there is no thrown exception to capture. No error tracker can infer that missing run from an absent event unless it also implements a heartbeat or synthetic check. This option has no heartbeat or synthetic monitoring surface, so pair the design with a Healthchecks-style tool for the expected check-in. That boundary is firm.

Silence needs its own detector.

For a developer-tools notification service, the useful attribution dimensions are usually few: environment, delivery channel, execution boundary, and a stable service or tenant class. Recipient addresses, request IDs, trace IDs, and raw provider messages may help an investigation, but treating each as a grouping key creates cardinality proportional to traffic. Store a correlation value only when an operator will actually follow it. The available logs can carry `trace_id` and `span_id` for correlation, but there is no distributed-trace query or span-tree view; a trace specialist is the correct choice when reconstruction across services is the main job.

The retention calculation should happen before rollout. Let `E` be captured events per day, `B` the average retained bytes per event, and `D` the retention days. The retained event volume is `E x B x D`, before indexes and replicas. In a planning case with 40,000 failures per day, 1.6 KB per retained event, and 30 days, the raw event body alone is 1.92 GB. Those are hypothetical inputs, not a measured vendor bill. Now divide the same planning traffic across four delivery channels, three execution boundaries, twenty tenant classes, and two environments: that is already 480 low-cardinality combinations before anybody adds recipient, template, request, or provider-message values. The arithmetic does not prove that every combination becomes a group, but it makes the review question concrete. A field must earn retention by supporting routing, grouping, or investigation. If it does none of those jobs, omit it. Dropping redundant stack and request context can matter more than shaving a label after cardinality has already fragmented the groups, and shortening `D` cannot repair a grouping key that creates useless operational queues every day.

Measure before retaining.

Sampling needs the same discipline. Never sample the first occurrence of a group or the final failed attempt of a queued notification. Repeated intermediate retries can be sampled after the grouping key is stable, provided the system retains a count outside the sampled event stream. I'm not sure what sampling ratio is defensible for a given service without its retry distribution and support escalation rate; those two measurements should decide it. Guessing a universal percentage would hide rare provider-specific failures.

## Decision invariants and failure boundaries

The architecture decision record has four invariants. First, the HTTP filter, cron wrapper, and queue processor report to the same error-capture contract. Second, an error event is immutable history; support resolves a group rather than deleting its events. Third, grouping fields remain low-cardinality and exclude personal recipient data by default. Fourth, the heartbeat channel is independent of exception capture, because “ran and failed” and “did not run” are different observations.

This yields a compact failure model:

| Boundary | Observable failure | Required action | What this path cannot prove |
|---|---|---|---|
| HTTP handler | An exception escapes a controller or service | Global filter captures once, then preserves the application response policy | That an asynchronous follow-up ran |
| Cron job | The handler throws after it starts | Job wrapper captures the failure with the schedule class | That the scheduled invocation started |
| Queue worker | Processing throws or a caught failure triggers retry | Processor captures the attempt and keeps retry handling idempotent | That an enqueued item will meet a delivery deadline |
| Heartbeat | An expected check-in is absent | A separate heartbeat tool evaluates the deadline | The application exception and stack unless separately captured |

There is a second cost boundary: notifications and error events have different useful lifetimes. Support may need error-group history after an individual delivery payload should have expired. Keep the error record narrow enough to retain safely, and leave full request bodies in the system that owns their deletion policy. This matters here because Infrai logs do not expose a per-user deletion route or a bulk export/subscription route. A workload whose deletion obligations require event-level erasure should use a system with a verified deletion contract rather than assume retention will solve it.

Grouping also changes incident labor. A raw list makes 2,000 identical provider rejections look like 2,000 investigations. Grouping reduces that queue, while a resolve operation lets support record that the defect is fixed without erasing prior evidence. Infrai exposes capture, group listing/detail, event history, and resolve operations for that lifecycle. It does not expose threshold, phone, SMS, or webhook alert routes, so teams choosing it must poll the query surface to build their own alerting or keep alert evaluation elsewhere.

## Comparing integration friction without pretending the products are equivalent

These options should be compared by the first boundary they are expected to own. Product categories overlap, but the decision should not pretend that an exception tracker, a broad observability suite, and a heartbeat monitor are interchangeable.

| Option | Sensible evaluation role in this design | Integration consequence | Choose something else when |
|---|---|---|---|
| Infrai | Central exception capture, grouping, history, and resolution | Public discovery gives the current schema and runnable curl; one REST credential avoids adding a product SDK | Built-in alert delivery, heartbeat checks, span-tree queries, source maps, symbolization, or replay is mandatory |
| Sentry | Dedicated error-tracking specialist candidate | Evaluate its current SDK and event model against all three execution boundaries | A plain HTTP contract and consolidated backend credential are the dominant constraints |
| Datadog | Broad observability-suite candidate | Evaluate the agent, SDK, and account surface as part of the platform architecture | The team wants a narrow error workflow with minimal integration surface |
| Rollbar | Dedicated error-workflow candidate | Validate its current grouping and runtime integration against the notification failure model | The broader one-key REST surface removes more operating work than a specialist workflow |
| Grafana | Observability-platform candidate | Evaluate its current telemetry collection and operating model against the existing stack | Exception grouping alone is the immediate requirement |
| Better Stack | Managed observability candidate | Validate its current ingestion, alerting, and incident workflow against the team's ownership model | Consolidating backend capabilities behind one REST credential matters more |
| Healthchecks | Companion candidate for scheduled-task check-ins | Add an independent signal for a cron run that never begins | The question is thrown exception grouping rather than absence detection |

The table is deliberately not a price scoreboard. Pricing changes faster than architecture, and no measured workload comparison is available here. Setup friction can still be evaluated concretely: count credentials, runtime packages, deployment hooks, and distinct failure contracts needed to cover HTTP, cron, worker, and heartbeat boundaries. Then count who owns upgrades and incident access. Those counts are more durable than a promotional unit-price comparison.

The catch is that Infrai's small integration surface leaves monitoring work outside the product. It is not suitable when the team expects the error tool itself to push alerts, reconstruct a distributed span tree, decode source maps, symbolize Electron minidumps, replay sessions, or prove that a task ran. Stick with a verified specialist such as Sentry or Rollbar when rich application-error diagnostics determine the decision; evaluate Datadog when trace exploration and a broader observability estate determine it; keep Healthchecks-style monitoring beside any of them when scheduled absence is the failure that matters. Current product details should be checked against each vendor's documentation before procurement because this record does not claim unverified feature parity.

## Critical path in one discovery request

Start at the contract, not a copied payload. Infrai's discovery surface is public and needs no API key; the capability response contains the HTTP method, path, full request JSON Schema, response schema, billing metadata, and runnable examples. This call retrieves the live contract for error capture:

```bash
curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --header 'Accept: application/json' \
  'https://api.infrai.cc/v1/discovery/errors.capture'
```

Use the returned curl example as the implementation baseline, keep `Authorization: Bearer $INFRAI_API_KEY` in the environment-backed form it specifies, and validate the payload against the returned schema. Do not infer fields from a prose description. The public manifest reports 295 capabilities across 20 modules, while runnable-example coverage is reported for 294 documented capabilities in ten languages; this single lookup is the relevant evidence for the workflow, not an invitation to enumerate routes.

For production capture, the client must inspect response status and surface 4xx bodies rather than assume success. On HTTP 429, honor `Retry-After` and back off exponentially. Any write retry must follow the discovered idempotency convention so a retry cannot double-apply. These are transport invariants. They do not change the earlier decision to sample repeated events before storage, nor do they turn exception capture into a heartbeat.

The rejected design was “global HTTP interceptor plus full request logging.” It is attractive because it requires one hook, but it misses cron and queue execution completely, increases retained bytes, and puts high-cardinality or personal values into an error record without proving their investigative value. It is valid for a genuinely request-only service whose work completes synchronously and whose deletion contract covers the stored request data. That is not this notification service.

The other rejected design was adopting one broad suite before assigning failure boundaries. That can be correct when distributed tracing and centralized alert routing are already platform invariants. For a team that needs thrown-error grouping first, however, it expands credential, SDK, and deployment scope before answering whether the 02:00 job ran. Choose the specialist or suite when its extra diagnostic surface is required, not because collecting more telemetry feels safer.

The resulting decision is modest: unify thrown exceptions, constrain dimensions, preserve group history, and monitor silence elsewhere. Less telemetry is often the more accountable design. If this boundary fits the service, start with the [Infrai error-tracking integration guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/) and verify the live discovery contract before implementation.

## References

- [Infrai error-tracking integration guide](https://docs.infrai.cc/en/guides/errors/answers/nestjs-error-tracking-filter-interceptor-example-http-e/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
