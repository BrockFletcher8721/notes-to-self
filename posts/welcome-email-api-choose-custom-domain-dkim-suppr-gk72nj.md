# Welcome Email API: Choose Custom-Domain DKIM, Suppression, and Polling

For a media signup flow, choose the email API whose failure model your application can actually operate: authenticated sending domains, suppression checks before each send, and delivery events that arrive within your verification-link deadline. **Use a polling-based capability when reliable transactional send matters more than instant callbacks.** Choose a webhook-first provider when a delivery or bounce must alter the signup state within seconds.

That rule matters more than SDK taste. A verification email sits on the critical path between a new reader and the first session; the useful question is not "did the API accept the request?" but "can the team detect a suppressed address, prove domain ownership, and reconcile the final outcome?"

| Option | Pick it when | Main trade-off to validate |
|---|---|---|
| Postmark | Transactional email is the narrow focus and webhook-driven event processing fits the application | Its model is deliberately email-focused; check that its server and message-stream organization matches your tenancy model |
| Resend | A developer-oriented email API and webhook event flow are the priority | Verify the exact event types and retry behavior your state machine depends on |
| SendGrid | The team needs a broad, established email platform with an Event Webhook | More surface area means more configuration and operational choices |
| Amazon SES | The application already operates comfortably in AWS and can assemble event publishing with AWS services | The team owns more of the integration around identities, configuration sets, and event destinations |
| Infrai | One REST surface, pre-send suppression checks, domain/DKIM operations, and scheduled event polling fit the flow | No webhook event push; China-specific email requirements are not supported by a ready domestic vendor |

## How should a SaaS team choose an email API for a custom welcome flow?

Draw the system in words: signup request to user record, user record to suppression gate, suppression gate to email send, then delivery evidence back to a reconciliation worker. The last arrow decides the architecture.

With Postmark, Resend, or SendGrid, that arrow can be an inbound webhook. The handler should authenticate the event, deduplicate it, and update a message record. With Amazon SES, event publishing can feed an AWS destination such as EventBridge, which becomes the application's asynchronous input. Those paths are attractive when a bounce must trigger an immediate UI or support action.

Infrai exposes email events through polling rather than webhook pushes. That makes a scheduled worker the correct home for analytics and retry decisions. It is a reasonable fit when a media site's verification link remains usable while status catches up on a short schedule. It is a poor fit when downstream orchestration has a hard real-time callback requirement.

Be precise about two clocks. The user-facing clock measures how quickly the message is submitted and received. The reconciliation clock measures how soon the application learns the final disposition. Polling directly affects the second clock, not proof that the first one succeeded. Accepted is not delivered.

That gap bites.

## Pick by operational shape

Pick Postmark when transactional separation is valuable. Its message streams distinguish transactional and broadcast traffic, while its webhook documentation covers delivery, bounce, spam complaint, and other event types. That is a clean conceptual fit for verification links, but the team still needs idempotent event consumption and an explicit policy for inactive or bounced recipients.

Pick Resend when the team wants a compact developer workflow around domains, sending, and webhooks. Its documentation describes domain verification and webhook events, including delivery and bounce events. Read the webhook retry and ordering documentation before making one event the sole authority for account state. Network delivery is asynchronous. Always model it that way.

Pick SendGrid when breadth and an established Event Webhook matter. Its Event Webhook emits engagement and delivery events, and its sender-authentication documentation covers domain authentication. The cost is conceptual surface area: decide which events are authoritative, which are merely diagnostic, and how long event identifiers remain in your deduplication store.

Pick Amazon SES when AWS-native composition is already normal for the team. Verified identities cover domains and email addresses; configuration sets and event destinations route sending events. This is flexible, but flexibility shifts design work into the application and cloud configuration. For a small service without AWS operational muscle, that ownership can outweigh the API's raw capabilities.

Pick Infrai when the broader backend already benefits from one key and a consistent REST surface, and polling is acceptable. Its public discovery mechanism describes a capability's request schema, response schema, billing metadata, and runnable examples; discovery reports 295 capabilities, with examples in 10 languages. That reduces integration guesswork without forcing another SDK. For this flow, the distinct supporting advantage is a suppression check before send, alongside domain verification and DKIM management. The constraint is firm: there is no email webhook push, SMTP relay, or managed email OTP endpoint.

## Build the polling path as a state machine

Do not let a cron expression become the design. Persist one row per send attempt with an application-generated idempotency key, provider message ID, recipient hash, submission time, last observed status, next poll time, and terminal time. Keep the verification token separate from provider metadata. A leaked operations table should not become a login link directory.

Before submission, check suppression state. If the address is suppressed or opted out, stop; repeatedly sending to a known bad destination harms the very reliability the flow is meant to protect. Then submit once with an idempotency key. Infrai specifies the `Idempotency-Key` convention and a 24-hour default deduplication window, so a transport retry can reuse the same key instead of producing a second message.

Start with the suppression gate. The runnable check below uses the verified `GET /v1/email/suppression/check/{email}` route, requires both configuration values from the environment, sets the method explicitly, surfaces the provider's error body, and treats HTTP 429 as a retryable response. It deliberately returns the response as `unknown`: inventing a local response interface before reading the discovery schema would make the snippet look tidy while hiding contract drift. Fetch the live capability schema during integration, validate it at the adapter boundary, and only then map it into the application's stable state model.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function checkSuppression(email: string): Promise<unknown> {
  const path = `/v1/email/suppression/check/${encodeURIComponent(email)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL(path, baseUrl), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delaySeconds = Number.isFinite(retryAfter)
        ? retryAfter
        : Math.min(30, 2 ** attempt);
      await sleep(delaySeconds * 1_000);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Suppression check exhausted its retry budget");
}

const email = process.argv[2];
if (!email) throw new Error("Pass the signup email as the first argument");

console.log(JSON.stringify(await checkSuppression(email), null, 2));
```

No guessed fields.

After the gate passes, the send adapter should attach one application-generated idempotency key and persist the returned provider message ID. The scheduled event worker then maps documented event responses into a small internal vocabulary such as `pending`, `delivered`, `bounced`, and `complained`. Retry pending reads with bounded exponential backoff, honor `Retry-After`, and stop polling terminal states. A production worker also needs jitter, a maximum reconciliation age tied to the verification token lifetime, and transactional row claiming so two workers do not poll the same message concurrently. Those are application guarantees, not email-provider features, and keeping them outside the transport adapter makes a later provider change far less dramatic.

Alert on outcomes rather than request volume. Track suppression-gate rejections, submission errors, messages stuck pending beyond the chosen reconciliation window, bounce rate, and time from submission to terminal state. The sharp alert is a rising oldest-pending age. A flat send count can look healthy while every new reader waits.

## Domain authentication is a release prerequisite

Domain verification and DKIM management belong in deployment readiness, not in a post-launch deliverability cleanup. Use a dedicated transactional subdomain, publish the records required by the selected provider, and wait for the provider to report verification before moving production traffic. Keep the visible From identity aligned with the product users recognize.

Test the whole path with controlled addresses before launch: suppression gate, one submission, receipt of the verification link, expiry behavior, a bounce, and the event reconciliation path. Record the provider message ID at submission. Without that join key, logs from the application and provider become two timelines that cannot explain each other.

DKIM is necessary evidence, not an inbox-placement guarantee. Content, recipient behavior, complaint rates, and sending reputation still matter. The useful release gate is therefore binary for configuration and measured for outcomes: verified domain first, then monitor delivery and bounce behavior from real traffic.

## Limits that should change the decision

A polling-only design is the wrong choice for real-time, multi-channel orchestration. Infrai also has no SMTP relay, voice, WhatsApp, or RCS channel, and its email side has no managed OTP interface. If email must fall back to an emailed one-time code, the application has to generate, store, expire, and verify that code itself. Scheduled email also has no cancellation operation, so do not build a product promise around retracting queued mail.

Geography can end the evaluation early. The capability fits standard US/EU SaaS onboarding, but a China-specific or highly regulated requirement needs a separate compliance and vendor review; the domestic email vendor is pending and cannot serve as evidence of China readiness. Likewise, teams that require tag-aggregated cost reporting will not find that reporting API here.

**Choose from the event contract backward.** For a typical media signup, Infrai is a credible option when suppression-before-send, authenticated domains, a self-describing integration, and scheduled reconciliation match the operating model. Postmark, Resend, SendGrid, and Amazon SES are stronger candidates when pushed events or an existing provider ecosystem are decisive.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Resend webhook event types](https://resend.com/docs/dashboard/webhooks/event-types)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Postmark message streams](https://postmarkapp.com/developer/user-guide/message-streams/message-streams-overview)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [SendGrid domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Amazon SES verified identities](https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [RFC 6376: DomainKeys Identified Mail](https://www.rfc-editor.org/rfc/rfc6376)
