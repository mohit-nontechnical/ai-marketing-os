# Reliability

MohitOS earns trust the boring way: processes that restart themselves cleanly, a publish path with no laptop in it, an append-only audit trail, and two small observability surfaces tied to real decisions. Most of what is documented here was built in response to a specific failure.

## Self-healing bridges: crash on purpose

The Telegram bridges are long-polling processes, and long-pollers fail in an ugly way: the process stays alive while its network loop limps, so a supervisor's keep-alive never fires. That exact failure once left phone approvals dead for about 33 minutes after a laptop sleep.

The fix is a watchdog that turns a limping process into a dead one:

```
consecutiveFailures = 0
loop:
  ok = poll()
  if ok: consecutiveFailures = 0
  else:
    consecutiveFailures += 1
    if consecutiveFailures >= 12:   # about 60 seconds of failures
      process.exit(1)               # let the supervisor restart us clean
```

Twelve consecutive poll failures (about a minute) and the bridge exits nonzero, getting a fresh process and a fresh connection from the supervisor (originally launchd on the Mac, now systemd with `Restart=always` on the droplet). Any successful poll resets the counter. Exiting is the recovery mechanism; the process never tries to nurse a wedged socket back to health.

## Getting the loop off the laptop

The deeper diagnosis behind most early instability: the Mac was in the critical path. Keep-awake hacks only assert on AC power, so approvals died whenever the lid closed on battery. The fixes came in two moves:

1. **Bridges moved to a $6/month DigitalOcean droplet** under systemd. The heavy worker (rendering, browser automation) stays on the Mac; the queue lives in cloud Postgres, so the droplet enqueues and the Mac drains jobs whenever it is awake. One rule survived the move: exactly one poller per bot token, ever, or Telegram returns conflict errors (the old Mac agents were disabled, not just stopped).
2. **The publish path was made Mac-free.** The bridge originally fetched the approval queue from the studio's HTTP API, which only exists on the laptop. That dependency silently killed approval cards for days when the bridge moved to the cloud (see the postmortem below). The rebuild: the bridge reads the queue from Postgres directly, publishes to Postiz in-process on approval, and media is pre-staged to the publisher at caption time so the droplet never needs a file from the laptop's disk. A cloud-readiness gate only pushes an approval card once the media is staged, so approving with the Mac off always succeeds.

The proof point: a post was approved by phone and published to Instagram entirely from the droplet, with the Mac not involved.

### The 12-day stall (the incident that paid for all this)

Between June 15 and 27 the pipeline was silently dark: approval cards frozen, captioned backlog climbing past 100. Two compounding root causes: a database migration that was written but never applied to the live schema, and one unset environment variable on the droplet that left the bridge pointing at localhost. Neither failure was loud. The lessons became infrastructure: verify live schema instead of trusting migration files, treat unset env on a new box as a deploy blocker, and build the observability below so silence itself becomes visible.

## The audit trail

`audit_events` is a single append-only table (append-only enforced by a database trigger) that records cross-cutting actions with a `trace_id` threading each flow across processes:

- A trace is minted at a flow's entry (a job, a request, a user action). Handlers wrap their body in `withTrace()` once; every nested `audit()` call, including across HTTP boundaries to external services, inherits the trace automatically via `AsyncLocalStorage`. No signature churn through the call stack.
- One query (`where trace_id = ... order by ts`) reconstructs the full filmstrip of a flow: which job fired, which service calls it made, latency, and outcome (ok, fail, or blocked).
- The helper is best-effort by construction: `audit()` never throws and never blocks the caller. An audit failure must not take down a content job.
- Two hard rules: never log secrets or tokens into the detail payload, and the module stays importable by both the web app and the queue worker.

It exists because the boundary between MohitOS and an external creative service was a black box; instrumenting the shared fetch wrapper in one place covered every endpoint at once. It also serves compliance: decision points on finance-adjacent content are logged with outcomes, which makes "why did this ship" answerable after the fact. Deliberately no UI viewer yet; SQL is enough until an investigation proves otherwise.

## Status observability

Two surfaces, each tied to a recurring decision, and a deliberate rule against growing more dashboards:

- **The worker status pill** sits in the studio's top bar and polls a status endpoint every 15 seconds: worker state (alive, stale, idle), running and queued counts, failures in the last hour, and the most recent completed job. Color rules are strict (red means the queue is stuck or failing repeatedly; amber means one recent failure or waiting work). Its best feature is a "Copy diagnostic" button that puts a compact text summary of worker state and recent failures on the clipboard, designed to be pasted straight into a Claude session. That is the standard first move when something looks broken.
- **The ops health strip** on the home page surfaces three slower-cadence signals: dispatch success and failure by target, approval queue depth and age (amber at 48 hours, red at 7 days), and attribution coverage. Those three exist because each answers a specific "do I need to act now" question; generic dashboards were explicitly rejected as tech debt.

The operating rule: when new observability is needed, extend one of these surfaces. A new page requires a new recurring decision to justify it.
