# Receipt Template Ownership: Safe Email Bounce and Complaint Handling Through Event Polling

| Template ownership | Pick this when | Operational obligation |
| --- | --- | --- |
| Application repository | Receipt copy changes with payment behavior | Application reviewers own content and event policy together |
| Shared template workspace | Content reviewers need a separate publishing workflow | Bind every send to an approved, immutable template revision |

Template editing freedom creates a release boundary: for a healthtech order receipt sent after payment settles, choose ownership that makes approval and rollback explicit. TL;DR: keep bounce and complaint policy outside the template editor. A Node.js polling job should consume normalized email events, record each event once, add persistent suppressions transactionally, and advance its checkpoint only after those writes succeed. Both ownership models can use that same control loop.

The payment settlement is the receipt trigger. A delivery event must never become another payment trigger.

## Who owns the receipt when its destination is suppressed?

Start with the responsibility that is easiest to lose between teams. A content owner can approve the receipt wording; that approval does not authorize another attempt to a destination that has complained. Treat suppression as an application policy, checked when the receipt is queued and again when a worker is ready to send it.

For this design, choose permanent delivery failures and complaints as reasons to suppress the destination within the sending account. Treat temporary failures as evidence for a bounded retry policy, rather than as permission to resend indefinitely. These are explicit application policies. The event adapter must document which source events satisfy each definition; a provider-specific label alone is insufficient.

Keep the scope visible. A suppression for one email destination does not cancel the order, reverse settlement, or automatically block every contact method for that patient. Any alternate receipt workflow needs its own approval and address-verification rules.

Draw the flow in words: payment settles → receipt intent records the approved template revision → sender checks suppression → transport accepts the attempt → event poller records feedback → later sends consult the updated suppression record.

That last arrow matters.

The receipt intent should carry an order reference and a template revision. Operational events need an opaque recipient key and a correlation key, not a rendered receipt or a patient name. Keep the destination lookup behind a separate interface so logs can explain a delivery decision without carrying the message body.

## Pick application-owned templates when payment behavior drives the copy

Choose repository ownership when a change to receipt wording commonly accompanies a change to settlement logic. Review the event that creates the receipt, its template revision, and its tests in the same change. This puts the operational contract close to the code that enforces it.

The trade-off is release friction. Even a small wording correction follows the application deployment process. That can be acceptable for a narrowly scoped receipt, especially when reviewers need to verify which payment state the wording describes.

Make a preview fixture deliberately plain: order reference `order-example-42`, payment state `settled`, template revision `receipt-v3`. Those are illustrative values, not production records. Check that the preview says what the settlement event actually establishes and that it excludes unnecessary health details.

## Pick shared ownership when content approval needs its own workflow

Choose a shared template workspace when designated content reviewers need to publish independently. The application should select an approved revision explicitly and store that selection with the receipt intent. Avoid resolving a mutable “latest” revision each time an old intent retries: otherwise a delivery retry can silently become a content change.

This model moves work into the publishing boundary. Require preview validation, approval evidence, and an explicit revision selection before a send is eligible. A rollback selects a prior approved revision for future intents; it does not need to rewrite the history of receipts already attempted.

Neither model should let a template publish clear a complaint suppression. Keep that authority separate. If the organization cannot name who owns reinstatement review, resolve that gap before granting editors a way to request repeat sends.

## How can an email polling job make bounce and complaint event handling safe?

Here is the useful implementation boundary: the source adapter lists events; the database decides whether their effects have already been applied. The TypeScript example is a portable contract for a Node.js job. It does not claim that any service exposes these exact fields or pagination semantics.

Require the adapter to provide stable event IDs within a stream and an opaque checkpoint suitable for resuming enumeration. Include the sending account in the stream identity. Normalize event meanings before they enter this function. If the source cannot support resumable enumeration or replay, establish a separate reconciliation method before calling this design complete.

```ts
type Kind =
  | "accepted"
  | "delivered"
  | "temporary_failure"
  | "permanent_failure"
  | "complaint"
  | "other";

type EmailEvent = {
  id: string;
  recipientKey: string;
  messageKey: string;
  templateRevision: string | null;
  kind: Kind;
};

type Page = {
  events: readonly EmailEvent[];
  nextCursor: string;
};

interface EventSource {
  listEvents(input: {
    stream: string;
    cursor: string | null;
    limit: number;
  }): Promise<Page>;
}

interface Transaction {
  lockCheckpoint(stream: string): Promise<string | null>;
  insertEventOnce(
    stream: string,
    event: EmailEvent
  ): Promise<boolean>;
  addSuppression(input: {
    stream: string;
    recipientKey: string;
    reason: "permanent_failure" | "complaint";
    eventId: string;
  }): Promise<void>;
  setCheckpoint(stream: string, cursor: string): Promise<void>;
}

interface Database {
  readCheckpoint(stream: string): Promise<string | null>;
  transaction<T>(
    work: (tx: Transaction) => Promise<T>
  ): Promise<T>;
}

type PollResult =
  | { status: "committed"; inserted: number }
  | { status: "checkpoint_changed"; inserted: 0 };

export async function pollOnce(
  source: EventSource,
  db: Database,
  stream: string
): Promise<PollResult> {
  const cursor = await db.readCheckpoint(stream);
  const page = await source.listEvents({
    stream,
    cursor,
    limit: 100
  });

  return db.transaction(async (tx): Promise<PollResult> => {
    const current = await tx.lockCheckpoint(stream);
    if (current !== cursor) {
      return { status: "checkpoint_changed", inserted: 0 };
    }

    let inserted = 0;
    for (const event of page.events) {
      if (!(await tx.insertEventOnce(stream, event))) continue;
      inserted += 1;

      if (
        event.kind === "permanent_failure" ||
        event.kind === "complaint"
      ) {
        await tx.addSuppression({
          stream,
          recipientKey: event.recipientKey,
          reason: event.kind,
          eventId: event.id
        });
      }
    }

    await tx.setCheckpoint(stream, page.nextCursor);
    return { status: "committed", inserted };
  });
}
```

The database methods carry real obligations. Enforce uniqueness on the pair `(stream, event.id)`. Make suppression insertion idempotent for a recipient and reason within that stream, preserving the triggering evidence. The checkpoint lock must serialize competing transactions for the same stream, including initial creation of the checkpoint row. Every method inside the callback must use the same database transaction; an exception must roll back the event ledger, suppression changes, and checkpoint together. Now the crash case has a clear answer. If the worker stops before commit, the next invocation reads the old checkpoint and replays the page. If commit succeeded but the process lost the response, the next invocation resumes from the committed checkpoint. Duplicate events on overlapping pages encounter the unique event key and produce no additional suppression effect. An older `delivered` event must not erase a newer complaint. This implementation never clears a suppression from a delivery event, so arrival order cannot cause that particular mistake. Reinstatement belongs in a separate, reviewed operation with its own evidence.

The batch limit of 100 is an example configuration, not a measured optimum or an API guarantee. Reduce it if transaction duration becomes a problem. Preserve the all-or-nothing boundary when tuning it.

Retry the poll, not the receipt.

A source read error should leave the checkpoint untouched. Let the scheduler retry with bounded backoff, honoring any documented server retry guidance; alert when the failure budget is exhausted. A checkpoint conflict is ordinary concurrency: return and let a later invocation read fresh state. Neither case should call the receipt sender.

## Observe decisions, then test the uncomfortable boundaries

Make a successful poll distinguishable from an empty poll. Record poll outcome, last successful completion, fetched event count, newly inserted count, and checkpoint conflicts. A high duplicate count can indicate normal overlap or a stuck enumeration path; inspect checkpoint progress before escalating. Poll completion alone says nothing about how recently the source made its events available.

For receipt investigation, use the correlation key to connect settlement, receipt intent, send attempt, template revision, and feedback. Keep recipient keys out of metric labels; place access-controlled correlation fields in structured logs instead. Track suppression decisions separately from delivery attempts so a rise in blocked receipts does not disappear into a generic “send failed” counter.

Set alert thresholds from the team's receipt-delivery objective and source behavior. No universal polling interval follows from this architecture. Evaluate request volume, transaction duration, source retention, and the maximum acceptable delay before a complaint blocks a future send. Cost enters as a workload constraint here: smaller polling intervals create more list requests, including empty ones.

Test the points where state can split. Replay a complaint event twice and verify one ledger entry and one effective suppression. Force an exception after the ledger insertion and verify that the checkpoint and suppression both remain uncommitted. Run two workers from the same checkpoint and verify serialization. Then deliver a complaint followed by an older delivery event; the destination must remain suppressed.

For deployment, exercise the adapter and transaction implementation with synthetic fixtures before enabling delivery side effects. Validate that the intended sending account maps to the intended stream. A perfectly deduplicated event from the wrong account is still the wrong decision.

Template ownership gets a test, too: change the approved revision while an intent is waiting. Verify that retrying the intent retains its recorded revision. This is where a copy-edit workflow becomes an observable delivery contract.

## Limits: polling cannot close every delivery gap

Suppression becomes effective after feedback is available and processed. A complaint can arrive between the sender's final check and transport submission. Rechecking narrows that window; it cannot undo a message already accepted for sending. Define the acceptable delay and investigate it with timestamps rather than promising immediate enforcement.

The ledger protects the feedback transaction. It does not make outbound receipt submission exactly once. Treat send-attempt identity and recovery after an ambiguous transport response as a separate design problem, and do not automatically repeat a send just because the client did not receive an acknowledgment.

RFC 8058 specifies a one-click unsubscribe mechanism using message headers and a POST request. That mechanism is distinct from polling complaint feedback; deciding which message classes need unsubscribe behavior requires a separate policy. The OWASP password-reset guidance addresses token and recovery security. It cannot establish receipt-delivery success, consent, or health-data compliance.

Keep those boundaries explicit. Template ownership determines who can change the receipt; the event ledger makes the consequences of delivery feedback inspectable.

## References

## Sources

- RFC 8058, One-Click Unsubscribe: https://datatracker.ietf.org/doc/html/rfc8058
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
