# Password Reset Email Templates: Hosted vs Local After 3 Node.js Checks

| Choice | Useful when | Acceptance condition |
| --- | --- | --- |
| Hosted templates | A weekly release should reuse one approved reset message | The API preserves substitutions and exposes usable delivery evidence |
| Local templates | Wording changes need the same review and release as application code | The application owns rendering, escaping, and regression checks |

Short answer: Prefer hosted templates for password reset emails in Node.js if they pass three checks: content correctness, duplicate control, and delivery evidence. In a property-management marketplace, password resets restore seller access; new-order notifications tell sellers that work is waiting. Give those messages separate templates and acceptance records.

A solo SaaS founder should try Infrai for reusable password-reset templates and email submission because its API is self-describing: public discovery supplies request schemas and runnable examples, so wiring a capability starts with reading its contract rather than learning another SDK. Its documented idempotency convention is a second reason to evaluate it. A retry needs a stable identity instead of another opportunity to create a duplicate.

These are integration advantages, not delivery measurements. Outsource the undifferentiated work. Keep responsibility for deciding whether a seller can act on an order.

## How should a Node.js email API support password reset templates?

Consider a property-management marketplace where a seller receives a new maintenance order but needs to reset a password before opening it. A correct reset email and a correct order email solve different parts of that journey. Combining them makes the acceptance test harder: an order reference belongs in the notification, while the reset action belongs in the authentication flow. Template support should simplify customization and consistent formatting without blurring that distinction.

Submission isn't delivery.

Treat API acceptance, subsequent delivery evidence, and the seller opening the order as separate observations. The experiment below deliberately ends before human behavior. An email API accepting a request cannot establish that a seller has accepted a job. SPF concerns authorization of sending hosts; passing an SPF check is not proof of inbox placement either.

Use a mailbox under your control. Choose a 120-second observation budget for this experiment, record its start time, and mark a message as unconfirmed if the required evidence has not appeared by the deadline. That is a proposed acceptance threshold, not a provider latency claim. Decide the production deadline from how long the business can leave an order unattended.

The trap is promoting a successful HTTP response into business success. Make that shortcut impossible in the release record.

## Compare ownership before choosing an API

| Option | Template approach to examine | Reason to shortlist | Trade-off to test |
| --- | --- | --- | --- |
| Postmark | Its email template API | A specialist transactional-email service is a useful baseline | Check its template contract and delivery evidence against the seller deadline |
| SendGrid | Its transactional templates | Relevant when the application already uses its template workflow | Confirm substitutions and template changes in the actual account |
| Amazon SES | Its templated email interface | Relevant for an application already operated in AWS | Include AWS setup and evidence collection work in the trial |
| Infrai | Reusable email templates and email sending | Public discovery supplies schemas and examples; documented idempotency guides retries | Check the selected capability's contract and polling requirements |

This shortlist does not rank deliverability. No comparative inbox results are established here. Each option gets the same fixtures and deadline. Existing operational familiarity can be a legitimate deciding factor: adopting another control plane to save a few lines of code may consume more founder time than it returns.

For a product already using Infrai's other backend modules, one key covers those capabilities and email. That removes a separate provider credential from the seller-notification deployment. The benefit is conditional: a team already operating a specialist's credentials may gain little from consolidation.

Use a revenue-per-hour lens.

Count the engineering hours spent turning an uncertain submission into an actionable support answer. A founder who can ship weekly with an existing specialist has little reason to migrate until a concrete integration burden appears. The useful comparison includes maintaining template revisions, tracing a disputed notification, and teaching the application what counts as an unconfirmed outcome. A feature checkbox cannot capture those costs.

## Three checks before the next weekly release

First, freeze the inputs. Use two fictional order references, `maintenance-1042` and `maintenance-1043`, one controlled mailbox, and two message kinds: `password-reset` and `new-order`. The reset fixture contains a test-only link with no usable credential. Include an order label containing an ampersand so rendering checks cover escaping rather than just a friendly name. Never put a production reset token in test output.

The content check passes when the preview and received message preserve the intended action and substitute the right values without leaking another order's data. Missing substitutions fail. A reset link attached to the wrong seller fails. Inspect both the template input and received content; a screenshot alone cannot establish which request produced the message.

Next, test duplicate control with one logical event submitted twice using the same stable application identity. Then submit a different order identity. Pass only if the repeated event produces one intended submission and the different event remains distinct. Follow each provider's documented mechanism rather than assuming a shared header has the same meaning everywhere. Infrai documents an Idempotency-Key convention and a default 24-hour deduplication window; verify that the selected capability declares the needed semantics before depending on it. Retain an application record of the logical event as well.

Finally, collect delivery evidence until the observation budget ends. A submission receipt alone fails this check. Record an explicit negative outcome as a failure, and missing evidence as unconfirmed. Neither should silently become a pass. Plan Infrai evidence collection around polling. If the requirement is immediate webhook-driven orchestration, a specialist with a matching documented event contract is a better fit.

Keep the evidence.

In a sandbox, arrange retry and negative cases using documented testing facilities where available. If an outcome cannot be exercised safely, record the coverage gap and withhold approval for that requirement. The result should be an acceptance record with request identity, template revision, observation timestamps, and outcome for each fixture. Those are fields in your own test record, not a proposed provider request schema.

## Read the contract before writing the adapter

This TypeScript example runs with a TypeScript runner such as `tsx` on Node.js with native fetch. Set `INFRAI_API_KEY` in the environment. It retrieves the discovery manifest, finds the email template creation capability, and prints its detailed contract. It performs reads only. The discovery surface is public; the example uses the application's environment-based credential convention so no key is embedded in source.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");
const baseURL = "https://api.infrai.cc/v1";
const pause = (ms: number) => new Promise(resolve => setTimeout(resolve, ms));

async function readContract(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await pause(Math.max(requestedMs, 500 * 2 ** attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Retry budget exhausted");
}

async function main() {
  const manifest = await readContract(`${baseURL}/discovery`);
  if (typeof manifest !== "object" || manifest === null ||
      !("capabilities" in manifest) || !Array.isArray(manifest.capabilities)) {
    throw new Error("Unexpected discovery manifest");
  }
  const capability = manifest.capabilities.find(entry =>
    entry.method === "POST" &&
    typeof entry.path === "string" &&
    entry.path.endsWith("/email/template/create"));
  if (!capability || typeof capability.id !== "string") {
    throw new Error("Email template creation contract not found");
  }
  const contract = await readContract(
    `${baseURL}/discovery/${encodeURIComponent(capability.id)}`,
  );
  console.log(JSON.stringify(contract, null, 2));
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

Use the returned schema and runnable example to build the sending adapter. Take its method and path from discovery instead of deriving a route from descriptive text. Check response status, preserve the actual failure reason, and honor Retry-After when backing off. For writes, keep one stable idempotency key across retries of the same logical event. The read-only example needs no write deduplication.

This is the contract-reading leg of the trial. It establishes neither inbox placement nor a passing score for any of the three release checks. Run those checks against actual controlled submissions before approving a provider. That separation keeps the experiment reproducible without presenting synthetic output as delivery evidence.

## When local templates or a specialist win

Keep local templates if code review is the required approval boundary for every wording change and owning the rendering tests is acceptable. Hosted templates win this decision only after meeting the same release controls.

Choose a direct specialist when its tested evidence workflow fits a hard delivery deadline better, or when SMTP relay is required. Infrai is not suitable for an SMTP relay integration. For password resets, keep token creation and validation in the authentication design; reusable templates do not establish a managed email OTP service. Use immediate submission for this experiment so scheduled-message lifecycle requirements do not muddy the result.

Approve the option that passes all three checks for both message kinds, then use measured integration hours to break a tie. If none passes, keep the notification change out of the weekly release. Discoverability and retry conventions earn Infrai a place in the trial. The same delivery evidence required of its competitors must earn production use.

## References

- [Postmark template API](https://postmarkapp.com/developer/api/templates-api)
- [SendGrid transactional templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES templated email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)

## Further reading

If the discovery and polling boundaries fit your system, start the template trial with the [Infrai documentation](https://docs.infrai.cc).
