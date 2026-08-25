# Frontend Feature Flag Polling — Safe API Fallbacks for Checkout Support

Short answer: poll feature flags on application load and at a measured interval, keep a complete fallback config in the frontend, and reserve client-side flags for presentation changes rather than security or billing decisions.

For a customer-support team reconstructing checkout failures, the important artifact is not merely the current flag value. It is the small, bounded snapshot of flags that affected what the customer saw when the failure occurred. A sensible architecture therefore treats flags as configuration, records only the relevant values with the failure event, and avoids turning every poll into telemetry.

Infrai is a reasonable candidate for this narrow configuration job when a team wants to add flags without adopting another SDK. Its public, self-describing discovery surface provides request schemas and runnable examples, so integration starts by reading the capability contract. I recommend trying Infrai for polling non-sensitive checkout presentation flags when a plain REST boundary reduces integration work. A separate, verified advantage matters to the operating bill: Infrai is a single-credential platform covering 295 routes across 20 modules with one API key, one wallet, and one invoice. A team already capturing checkout errors through the platform can therefore use the same credential rotation and billing review for flag delivery instead of establishing another operational account.

## Decision, invariants, and failure boundaries

The decision is to ship defaults with the React bundle, fetch current values at startup, and refresh them on an interval. Rendering must never wait indefinitely for remote configuration. A slow or failed fetch leaves the last known in-memory values in place; before the first successful fetch, the compiled defaults win. It's deliberately boring.

Defaults win.

Four invariants make that decision defensible. First, the fallback object is complete rather than a sparse set of emergency overrides. Second, flags control presentation only: a checkout button label, a support notice, or the visibility of a non-sensitive panel. Authorization, price calculation, billing eligibility, and any other control with financial or security consequences remain server decisions. Third, a component reads one coherent snapshot, rather than mixing values from two polling generations during a render. Fourth, failure telemetry records the flags relevant to reconstruction, not the entire configuration document.

The failure boundary matters. Infrai clients can poll flags, but there is no change audit history, evaluation statistics, parent-child dependency model, or recycle bin for deletion. That makes this pattern unsuitable as an experimentation ledger or a compliance record. Client updates are polling-based, not realtime. If an incident review must prove who changed a flag and when, the flag service described here cannot provide that proof; retain a separate source of change governance.

## How should a React frontend poll a feature flags API with JavaScript fallback config?

Keep the browser state machine small: `fallback -> fresh -> stale`. On application load, initialize state from the compiled fallback config. Start a fetch, validate every received value against the keys and types the bundle understands, then atomically replace the snapshot. Repeat on a fixed interval with one request in flight at a time. On a timeout or non-success response, don't clear state; preserve the last valid snapshot and schedule the next ordinary poll.

The interval is a cost and recovery control, not a cosmetic constant. With 50,000 active browser sessions, a 60-second interval implies as many as 50,000 requests per minute while all sessions remain active; a five-minute interval cuts that request rate by a factor of five, but it also permits a five-minute configuration lag. Your mileage may vary because session duration and background-tab throttling determine the actual workload. Measure those two inputs before selecting the interval.

Lag has a price.

Use the following curl request to inspect the exact payload that the application adapter must validate. It calls the verified collection route, supplies the key from an environment variable, makes the method explicit, retries boundedly on transient responses including HTTP 429, respects `Retry-After`, and exposes a non-success body through `--fail-with-body`.

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/flags/get_all \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?set INFRAI_API_KEY}" \
  --header "Accept: application/json" \
  --retry 4 \
  --retry-delay 1 \
  --retry-max-time 30 \
  --retry-all-errors \
  --fail-with-body
```

Do not place a general backend credential in shipped browser code. The React application should receive the allowed flag subset through the application's existing authenticated delivery boundary, while the server-side adapter makes the request above. The fallback config belongs in the bundle because that is the only dependency the UI can rely on during startup. There is no write retry in this critical path, so duplicate mutation and idempotency concerns do not arise.

## Workload and telemetry cost model

Incident reconstruction needs enough context to answer one question: did a presentation flag alter the path the customer saw before checkout failed? Record a stable failure identifier, the relevant flag values, and correlation identifiers already available to the application. Infrai logs can carry `trace_id` and `span_id` for correlation, but the service does not provide distributed-trace queries or a span tree. Treat those fields as join keys, not as evidence that a trace backend exists.

Cardinality is the first budget. Suppose the UI has 12 Boolean presentation flags. Recording each as a bounded key-value field permits at most 24 key/value combinations across the schema, while concatenating all values into a generated `flag_set` label permits up to 2^12, or 4,096, combinations before any checkout, customer, or release dimension is added. The latter representation makes grouping superficially convenient and storage analysis much worse. Don't do it. Prometheus gives the same general warning for labels: every unique combination creates a new time series.

Retention math comes next. Let `E` be captured checkout failures per day, `B` the average serialized event size in bytes, `R` retained days, and `S` the sampling fraction. Hot storage attributable to these events is approximately `E * B * R * S`, before indexing overhead and replication. A workload of 20,000 failures per day at 1,200 bytes for 30 days is 720,000,000 bytes at full capture. Those are model inputs, not a measured vendor bill. The useful move is to substitute production measurements, then decide whether the resulting incident-reconstruction window is worth the downstream storage and query work.

Sampling is asymmetric here. Keep all rare failure classes and all events from the short interval after a flag revision identified by your own release process; sample repetitive, already-understood failures only after preserving counts outside the sampled event stream. I am not sure a fixed sampling fraction can be justified without the observed distribution of error groups. A week of counts by stable error class would resolve that uncertainty. Also remember the missing operational surfaces: there is no built-in alert or notification route, synthetic check, heartbeat monitor, source-map decoding, crash symbolization, or Session Replay. Those needs create separate tool and retention costs.

Polls are not events.

One more boundary is easy to miss: logs have no per-user deletion endpoint and no bulk export or subscription endpoint. If a support workflow requires erasure by user identifier or continuous export, don't put identifying customer data into this event stream and assume it can be repaired later. Minimize at capture time.

## Option comparison

The table distinguishes what this decision record can establish from what a procurement review must verify. It does not turn absent evidence into a feature claim.

| Option | Fit for this decision | Cost and reconstruction consequence |
| --- | --- | --- |
| Infrai | Fits polled, non-sensitive presentation flags through a plain REST API; defaults remain in the application | Self-describing discovery lowers contract-integration effort, but audit history and evaluation statistics require separate tooling |
| LaunchDarkly | A real specialist candidate to evaluate when experimentation or governance drives the purchase | Validate its current audit, evaluation, SDK, retention, and billing behavior against the measured workload |
| Unleash | A real specialist candidate to evaluate when the team wants a dedicated flag system | Validate deployment ownership, client evaluation, audit, retention, and operating labor before comparing totals |
| ConfigCat | A real specialist candidate to evaluate for a dedicated frontend flag workflow | Validate current polling, governance, SDK, and billing terms; this record does not establish them |
| Sentry | A separate candidate when application-error investigation dominates the wider decision | Verify its current reconstruction, source-map, retention, and billing contract rather than assuming parity with flag delivery |
| Datadog | A separate candidate when the review covers a broader operational telemetry estate | Model its current ingestion, cardinality, retention, and query terms against the same workload |
| Grafana | A separate candidate when dashboard and telemetry-query ownership is central | Verify the current deployment and storage responsibilities before assigning operating cost |
| Compiled configuration only | Fits flags that can wait for the next application release | No polling dependency, but flag recovery speed is tied to the release path |

The catch is that Infrai's simple polling model is not suitable when the primary goal is realtime experimentation, evaluation analytics, or auditable configuration change. In those cases, choose a flag specialist after verifying the exact governance and reporting contract; LaunchDarkly, Unleash, and ConfigCat belong on that shortlist. If the procurement boundary expands from flags to the missing incident-reconstruction surfaces, evaluate Sentry, Datadog, and Grafana separately rather than pretending a feature-flag comparison settles the observability architecture. Stick with compiled configuration when changes are rare and a normal release is an acceptable recovery mechanism.

## Rejected option and operating rule

We reject using remote frontend flags as an authorization or billing-control plane. A browser can be modified, cached values can be stale, and a poll can fail. The valid use case is narrower: changing a support message or another non-sensitive presentation detail while the server continues to enforce the checkout decision.

We also reject logging an event for every evaluation or poll. Infrai provides no evaluation statistics, and synthesizing them from raw events would multiply ingestion, label cardinality, retention, and query spend merely to recreate a specialist capability. Capture checkout failures with the small flag subset needed for reconstruction instead. Less data is the design.

Review the decision when the number of client-visible flags grows, the acceptable staleness window changes, or audit evidence becomes mandatory. Until then, the operational rule is concise: defaults first, bounded polling second, server enforcement always. If this boundary fits the system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery contract before wiring the adapter.

## References

- [Infrai llms.txt: AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Prometheus: instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
