# Regional failover testing for a shared realtime kanban board on Node.js and Postgres

Presence accuracy and regional failover pull against each other, and on a shared kanban board the tie-break decides your entire test strategy. Set the presence timeout to three seconds so the avatar row is honest, and a regional failover reads as forty people quitting at once. Set it to sixty seconds so failover stays invisible, and two dispatchers will edit the same work order because each still sees the other as gone. In short, the way out is to stop deriving presence from the socket and start issuing it as a lease with an explicit expiry — then use an injected clock so the tests can expire that lease on demand instead of sleeping.

**Presence is a lease with an expiry, and the test suite must own the clock that expires it.**

The system worth having in mind is unremarkable property management software: a shared kanban board where maintenance work orders move from Reported to Scheduled to In Progress to Done, with something like forty dispatchers and field technicians connected during business hours. Polling is not on the table here. Forty clients at a five-second interval is eight requests per second of pure overhead, most of them returning nothing new, and the avatar row is still up to five seconds stale — you pay the request budget and don't even get accuracy for it. So the board pushes over a WebSocket, and presence turns into the hardest part of the whole feature.

## Why presence accuracy and regional failover fight each other

A WebSocket tells you about a transport, not about a person. Connection state answers "is this session open"; the board needs "is this dispatcher looking at column three right now"; those two questions diverge the moment anything below the application layer moves. A regional failover moves everything below the application layer at once.

If presence is derived from socket state, evacuating a region is indistinguishable from a mass logout. Every socket closes inside one health-check window, the presence set empties, the board repaints forty avatars to grey, and a second later the clients reconnect to the standby and the set refills. Users see a flicker. Anything keyed on presence transitions sees a full spurious cycle: assignment handoff, the "someone else is editing this card" badge, idle reassignment of an urgent work order.

A lease breaks that coupling. The client asks for presence with a time-to-live; the server records session, board, and expiry in a store both regions can read; the client renews at roughly one third of the TTL. A dropped socket is then no longer evidence of absence — only missing renewals are. With a 30 second TTL and a 10 second renewal interval you tolerate two consecutive lost renewals, which covers a failover that completes inside about 20 seconds, and you still notice a closed laptop within 30.

The literature calls this separating failure detection from membership dissemination, and the SWIM protocol argues it better than I can.

Peer-to-peer transports are a tempting shortcut for the cursor layer, since a data channel skips the server round trip entirely. They don't produce a server-side lease, though, and a board whose authority lives in Postgres needs one record of who holds what. That makes them a poor fit as the presence substrate, whatever they do for cursors.

The catch is the store. Cross-region leases only work if the rows are visible to whichever region takes over — a Postgres table with an expiry column and logical replication, or a replicated key carrying its own TTL. If presence state lives in a single-region cache, this design gives you nothing during the exact event it was built for, and you'd be better off with socket-derived presence, a last_seen column, and a deliberately generous timeout.

## How do you test a shared kanban board across a regional failover without flaky timing?

Three failure modes matter, and each has a deterministic analogue that never needs a sleep.

| What happens in production | What to reproduce | Deterministic test |
| --- | --- | --- |
| Region evacuation | every socket closes inside one health-check window | close all sessions server-side in a single tick, then advance the injected clock past the TTL |
| Slow drain | sockets stay open, renewals stop arriving | block renewal frames at a fault-injection proxy and leave the connection up |
| Split view | a client reconnects to the standby while the old lease is still recorded | write the same lease twice with different fencing tokens, assert the higher token wins |

Every one of these is a state assertion rather than a timing assertion. The test drives the system to an observable condition — a lease row whose expiry is in the past, a broadcast the client acknowledged, a version counter that incremented — and only then asserts.

Nothing waits for elapsed wall-clock time, so nothing is flaky.

Fault injection covers the transport half of the problem. A proxy in front of the WebSocket endpoint can sever, delay, or partially drain connections on command, which is how you separate "the socket closed" from "the socket is open and useless". Those are different bugs with different fixes, and a suite that can only produce the first one will keep missing the second.

Here is the renewal the client repeats, in the form that survives being pasted into an incident channel:

```bash
curl -sS -X POST https://board.example.com/presence/lease/renew \
  -H "authorization: Bearer $BOARD_TOKEN" \
  -H "content-type: application/json" \
  -d '{"board":"north-portfolio","session":"s-8f21","ttl_seconds":30}'
```

The response carries the expiry the server will honor, and that is the value the test asserts against — never the value the client asked for.

```json
{"board":"north-portfolio","session":"s-8f21","expires_at":"2026-03-04T09:12:41Z","fence":184}
```

The test-only control plane, present in the staging build and compiled out of production, moves time instead of consuming it:

```bash
curl -sS -X POST https://board.test.internal/_test/clock/advance \
  -H "content-type: application/json" \
  -d '{"seconds":31}'

curl -sS https://board.test.internal/presence/board/north-portfolio
```

A suite built this way finishes in single-digit seconds and behaves the same on a loaded CI runner as on a laptop.

## The clock is an input, not an ambient fact

The single change that removes most timing flakiness is refusing to let the expiry evaluator read the ambient clock. Inject it. A `now()` dependency at the edge of the presence module means the same code path runs against a monotonic source in production and against a value the test moves in whole seconds in CI.

Two clocks, two jobs. Wall-clock time is for display — "last seen 09:12" — and it is allowed to jump when NTP corrects it. Monotonic time is for expiry, because a lease that expires early because the host stepped its clock backwards is the kind of defect you will never reproduce on demand.

The browser half deserves the same treatment. Current end-to-end runners can install a controllable clock inside the page, which puts reconnect backoff, renewal cadence, and the offline banner under test without real delays; the Page Visibility API is the other half of that story, since a backgrounded tab is throttled and will miss renewals a foreground tab would have sent. Test the throttled path explicitly, because on a shared board the throttled tab is the one whose owner walks away.

I'm not entirely sure there's a clean way to fake one thing: the TLS and HTTP upgrade handshake against the real load balancer. A fake clock hides handshake regressions, so keep a single real-time smoke test per release — two minutes, one connection, one forced failover — and accept it as the slowest test you own.

## Counting what presence costs in cardinality and retention

Presence is chatty by construction, and the bill lands in the observability system rather than the application one.

Do the arithmetic before shipping it.

Forty concurrent sessions renewing every 10 seconds produce 4 events per second. Over a ten-hour working day that is 144,000 renewal events; at roughly 350 bytes per structured log line, about 50 MB per day, or 1.5 GB per month per environment before replication and index overhead. Thirty-day retention on that raw stream is a real line item for a feature whose entire user-visible surface is a row of small circles.

Cardinality is worse than volume, and teams find it late. Label a presence metric with user, board, and state and you have created 180 staff accounts × 60 boards × 3 states = 32,400 active series; add session_id, which is unbounded by design, and the series count becomes a function of reconnect churn — exactly the number that spikes during the failover you were trying to observe. Keep the metric labels to board and state, which is 180 series, and push user and session into the log line, where high cardinality belongs.

Sampling is where the honest trade-off lives. Dropping 90% of renewal events uniformly cuts retention cost by an order of magnitude and destroys the one thing forensics needs, which is a complete timeline for a single session. Hash the session id instead: keep 100% of a stable 5% of sessions, plus 100% of every lease transition — issued, renewed late, expired, fenced. Transitions are rare next to renewals, and transitions are what reconstruct a failover.

Keep the sampled raw stream for 48 hours and the transitions for 30 days.

## Rolling the lease path out without a big-bang cutover

Ship the store first and read from it second. Write lease rows alongside the existing socket-derived presence for a week, compare the two sets on a dashboard, and only then flip the board to read leases behind a per-account flag. Rollback is one flag and a longer TTL, which matters because presence defects are visible to every user at the same moment.

Run the deterministic suite on every commit, and one real failover exercise a month against staging with the proxy performing the evacuation. A twenty-minute game day that produces a graph of lease expiries teaches more than another twelve unit tests.

**Not every board needs this.** Three people sharing a board from the same office are not a good fit for lease machinery — stick with socket-derived presence, a last_seen timestamp, and a sixty-second timeout, and spend the effort on conflict resolution for the cards themselves. The design earns its complexity when presence drives automation: auto-assignment, escalation timers, or anything that acts on the belief that a person has left.

## References

- RFC 6455, The WebSocket Protocol — https://www.rfc-editor.org/rfc/rfc6455
- W3C WebRTC Recommendation — https://www.w3.org/TR/webrtc/
- WHATWG HTML Standard, Server-Sent Events — https://html.spec.whatwg.org/multipage/server-sent-events.html
- MDN, Page Visibility API — https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API
- SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol — https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
- Playwright Clock API — https://playwright.dev/docs/api/class-clock
- Toxiproxy, a TCP proxy for simulating network conditions — https://github.com/Shopify/toxiproxy
- Prometheus, metric and label naming — https://prometheus.io/docs/practices/naming/
