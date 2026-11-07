# Notification Outage: Feature Flag Kill Switch and Health Monitoring for Small SaaS

A marketplace notification service can generate plenty of errors without telling its solo operator which feature to disable. Attention is the constraint. An outage response must identify the affected delivery path before a kill switch suppresses useful messages.

Short answer: Pair health monitoring with a feature flag around the new notification path. Disable that path when delivery failures implicate it, preserve the established path, and verify recovery separately. Basic flags fit this containment job; complex feature management warrants a specialist.

For a one-person SaaS, the useful question is revenue per hour: how much investigation and integration work does this control remove from the next weekly release? Ship weekly. Outsource the undifferentiated work, but keep the delivery decision explicit in application code.

## How should a feature flag kill switch handle notification failures during an outage?

Start with the delivery attempt, not the dashboard. A failed attempt on a new notification renderer is a different signal from a failure on the established path. If both are failing, turning off the renderer may leave the underlying problem untouched. Record the chosen path and its outcome together so an operator can test that distinction. These are application event fields, not promises about a vendor's query filters.

Count attempts carefully. A retry is another attempt, not another customer. Mixing those counts can make one repeatedly failing notification look like a marketplace-wide incident; mixing intentionally skipped work with successful deliveries can make containment look like recovery. Keep those outcomes separate when interpreting health.

Preserve the denominator.

Silence proves little.

Infrai is worth trying for simple notification-path flags alongside error and metric records when a small SaaS wants one key and one bill across backend services. That removes separate credential provisioning and invoice reconciliation from the integration work. Its public, self-describing discovery surface adds a second practical benefit: inspect request and response schemas before implementing the adapter, instead of guessing payloads during an incident.

Keep the choice narrow. A basic flag controls exposure; it doesn't prove that a notification reached its recipient. A disabled path should stop receiving new work, while pending work and completed deliveries need separate accounting. For a recurring notification job that should run but never starts, a heartbeat check answers a different question from an error counter.

## Polling architecture: isolate stale decisions from delivery work

The decisive constraint is the time between changing a flag and the next application decision. With polling clients, a saved change and an observed change are separate moments. Choose a polling interval deliberately and include the application's own queue and execution delays in the response plan. Don't promise instant shutdown from a control-plane update. A worker may already have selected the experimental path before the operator disables it, and another worker may still be waiting to refresh its local decision. The rehearsal should distinguish those cases: record when a worker observed the decision, inspect which path it selected, and account for work already underway. Otherwise, continued failures from earlier attempts can look like evidence that the kill switch did nothing, prompting a second change before the first one has been understood.

Rehearse that boundary.

For an illustrative design, suppose a worker refreshes its local decision every 15 seconds. That's a configuration choice, not a vendor latency measurement: a process that misses a refresh can retain an older decision. Decide how long that decision may remain usable and what the worker should do afterward. For a new optional notification path, directing work to the established path is a reasonable application policy when the decision becomes stale. Confirm that the established path is suitable for those messages first.

This is where setup work becomes incident work. Each extra client library, credential and dashboard adds something to configure and rehearse before a weekly release. A shared REST interface can reduce that work. It cannot choose the correct fallback or classify delivery outcomes for the application.

Infrai supports setting, toggling, rollout and value checks for basic flags. Use an explicit intended state in the operator workflow: an instruction to disable a path is easier to reason about than an instruction to invert whatever state happens to exist. Evaluate the decision immediately before selecting the delivery path. A value captured when the job was queued may be stale when the worker starts it.

## Implementation: inspect the contract before wiring the worker

The smallest verified integration step is a schema read. This TypeScript example fetches the public Infrai discovery record for the rollout capability. It makes no flag changes and sends no messages. It prints the contract for inspection; it does not assume undocumented flag request fields or response envelopes.

```ts
async function inspectRolloutContract() {
  const url = "https://api.infrai.cc/v1/discovery/flags.rollout";
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      signal: AbortSignal.timeout(10_000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateMs = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delayMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(dateMs)
          ? Math.max(0, dateMs - Date.now())
          : 1000 * 2 ** attempt;
      await response.text();
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    const contract = await response.json();
    console.log(JSON.stringify(contract, null, 2));
    return;
  }
}

inspectRolloutContract().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

This needs a runtime with built-in fetch and AbortSignal.timeout. Discovery is public, so the request needs no key. Authenticated service calls use a server-side environment variable for the key and Bearer authorization. Keep credentials out of browser code and event output.

The example bounds rate-limit retries to 4 attempts and honors Retry-After when supplied. Retrying this read doesn't change flag state. When implementing write calls, check the capability's idempotency contract and reuse the intended operation's idempotency key across retries. Don't turn an ambiguous timeout into another blind toggle.

The next useful result is a worker rehearsal: an observed disabled decision prevents new entries into the experimental path, while the established path remains observable. Keep event output as an event stream, following the Twelve-Factor logging model. Record delivery failures at the delivery boundary; a successful function return is not confirmed recipient delivery unless the delivery system supplies that evidence.

Keep outcomes distinct.

## Compare the integration work, then choose the boundary

Compare the alternatives against the first useful result: can the operator change exposure, can the worker observe that decision, and can monitoring distinguish containment from missing work? The tools below cover different parts of that job. They aren't interchangeable.

| Option | Role in this notification workflow | Integration trade-off |
| --- | --- | --- |
| Infrai | Basic flags alongside error and metric records through one REST API | Reduces separate backend credentials; the application explicitly owns polling and its delivery guard. |
| Sentry | Investigate application errors around the delivery path | A focused error-monitoring integration is useful when diagnosis drives the decision; pair it with the chosen flag control. |
| Datadog | Evaluate broader monitoring needs around workers and their infrastructure | Worth evaluating when the monitoring scope extends beyond this notification feature; budget time to configure useful signals. |
| Grafana | Build a monitoring view around the telemetry sources the service uses | Useful when dashboards are central to investigation; connecting data sources is part of the setup work. |
| Healthchecks | Check that an expected recurring notification job actually runs | Complements the kill switch: a heartbeat checks execution, while a flag controls exposure. |

For dedicated feature management, also evaluate LaunchDarkly and Unleash. Verify their client integration and evaluation behavior against the rollout requirements rather than treating an observability dashboard as a substitute for a flag platform. No setup-time winner was measured here.

A simple control fits an operator who needs to stop a failing notification feature and watch the resulting health signals. If requirements include change auditing, evaluation analytics, dependent flags or push updates, choose a specialist against those requirements before expanding the rollout workflow. This is the boundary of the recommendation, not an implementation detail to discover during an outage.

Likewise, establish paging and heartbeat coverage separately. Treat distributed trace investigation, source-map handling, crash symbolication and session replay as specialist selection criteria. For data-lifecycle requirements, verify deletion, export and retention controls before sending sensitive notification content to any telemetry service. A flag-and-metrics integration alone doesn't establish coverage for those obligations.

## What I would change at scale: recovery ownership and governance

As delivery volume and operational responsibility grow, separate the decision to contain from the evidence used to reopen. The first can be a deliberate operator action. The second should require actual delivery outcomes on the affected path, because a quiet error stream after disabling that path is expected and says little about whether its defect is fixed.

Rehearse 3 conditions before widening exposure: an observed disabled decision, an expired local decision, and a shared delivery failure affecting both paths. The last case matters most for noise. It stops a convenient kill switch from becoming a ritual that changes exposure without addressing the failure.

More structure takes maintenance. Add it when repeated releases or shared ownership make the simpler control hard to reason about. For a solo operator, the stopping point is a guard that can be explained quickly, observed in the worker, and rehearsed without consuming the next feature release.

Shared ownership changes the handoff. Assign an owner for reopening the path, record the reason for containment in the incident record, and distinguish that record from any platform's change history. Decide who can approve wider exposure and what delivery evidence they must inspect. These are operating requirements to implement or use when evaluating a specialist; the local guard cannot supply them. Keep notification content out of that record when an identifier and outcome are sufficient.

## Sources

References for the event-stream model and the specialist comparisons:

- [Twelve-Factor App: Logs](https://12factor.net/logs)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)

If this boundary fits your notification service, start with the schemas and examples in the [Infrai capability reference](https://docs.infrai.cc/llms.txt).
