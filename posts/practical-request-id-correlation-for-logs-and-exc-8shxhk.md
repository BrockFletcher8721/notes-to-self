# Practical Request ID Correlation for Logs and Exceptions Together During Pricing Rollouts

Express error tracking during a pricing-rule rollout fails when Pino or Winston logs cannot be joined to captured exceptions. Choose one request ID at the HTTP boundary, carry it through asynchronous work, and send it with both signals so an engineer can reconstruct a checkout without guessing.

**Short answer:** in an Express service, use `AsyncLocalStorage` for request-scoped context, enrich either Pino or Winston through a tiny adapter, and capture exceptions only in the final error middleware. The logger and exception tracker remain separate outputs; `request.id`, `flag.key`, `flag.variant`, and `pricing.rule_id` make them one searchable incident trail.

This is correlation, not duplication. Shipping every log line as an exception creates noise, while logging an `Error` without capturing its stack leaves the error tracker blind. Keep both signals. Give them the same join key.

One ID. Two outputs.

## Why does Express error tracking need Pino or Winston request ID correlation?

A checkout can cross flag evaluation, catalog lookup, discount calculation, tax calculation, and payment authorization before the response returns. If a newly enabled pricing rule changes a total from 49.00 to 44.10, the incident question is rarely just "what threw?" The useful question is "which rule and flag variant ran for the request that threw, and what happened immediately before it?"

The before model is fragmented: the access log has a generated ID, an application log has a message and perhaps an order ID, and the exception event has a stack trace. An operator searches by timestamp and hopes concurrent checkouts do not interleave. The after model is a chain in words: inbound request -> validated request ID -> asynchronous context -> child logger -> pricing decision -> exception capture. Every arrow carries the same `request.id`.

Short IDs are tempting. Don't use a process-local counter: two instances can both issue `42`, and a restart repeats it. The example accepts an inbound UUID or creates one with `crypto.randomUUID()`. It also returns the ID in `X-Request-ID`, which gives support staff and API clients a concrete lookup value. Node documents `AsyncLocalStorage` as stable and designed to keep data coherent through asynchronous operations; that is the mechanism doing the quiet work here.

One boundary matters. An inbound ID is untrusted input, so validate it and cap what enters telemetry. OWASP warns that logs can become a target for injection and that sensitive data such as access tokens, passwords, and payment-card data should normally be excluded. A UUID-only policy solves the first part for this field. A field allowlist and redaction policy must handle the rest.

## Build one correlation boundary

The copyable example below supports either logger without running both at once. That choice is deliberate. Pino and Winston have different APIs, but the application needs only `info` and `error`; forcing every route to understand both libraries spreads an operational decision across the codebase. The exception tracker is represented by a narrow interface because SDK names and initialization differ, while the correlation contract does not.

```ts
import { randomUUID } from "node:crypto";
import { AsyncLocalStorage } from "node:async_hooks";
import express, { NextFunction, Request, Response } from "express";
import pino from "pino";
import winston from "winston";

type Context = {
  requestId: string;
  flagKey?: string;
  flagVariant?: string;
  pricingRuleId?: string;
};

type Fields = Record<string, unknown>;

interface AppLogger {
  info(message: string, fields?: Fields): void;
  error(message: string, error: Error, fields?: Fields): void;
}

interface ExceptionTracker {
  captureException(error: Error, context: { tags: Fields; extra: Fields }): void;
}

const contextStore = new AsyncLocalStorage<Context>();
const uuidPattern =
  /^[0-9a-f]{8}-[0-9a-f]{4}-[1-8][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;

function currentFields(fields: Fields = {}): Fields {
  const context = contextStore.getStore();
  return {
    "request.id": context?.requestId,
    "flag.key": context?.flagKey,
    "flag.variant": context?.flagVariant,
    "pricing.rule_id": context?.pricingRuleId,
    ...fields,
  };
}

const pinoBase = pino({
  redact: ["req.headers.authorization", "customer.email", "payment.card"],
});

const pinoAdapter: AppLogger = {
  info(message, fields = {}) {
    pinoBase.info(currentFields(fields), message);
  },
  error(message, error, fields = {}) {
    pinoBase.error({ ...currentFields(fields), err: error }, message);
  },
};

const winstonBase = winston.createLogger({
  format: winston.format.json(),
  transports: [new winston.transports.Console()],
});

const winstonAdapter: AppLogger = {
  info(message, fields = {}) {
    winstonBase.info(message, currentFields(fields));
  },
  error(message, error, fields = {}) {
    winstonBase.error(message, { ...currentFields(fields), error });
  },
};

const logger = process.env.LOGGER === "winston" ? winstonAdapter : pinoAdapter;

// Initialize the real tracker during application startup.
const exceptionTracker: ExceptionTracker = {
  captureException(error, context) {
    process.stderr.write(
      JSON.stringify({
        type: "exception",
        error: { name: error.name, message: error.message, stack: error.stack },
        ...context,
      }) + "\n",
    );
  },
};

const app = express();
app.use(express.json());

app.use((req: Request, res: Response, next: NextFunction) => {
  const inbound = req.header("x-request-id");
  const requestId = inbound && uuidPattern.test(inbound) ? inbound : randomUUID();
  res.setHeader("X-Request-ID", requestId);
  contextStore.run({ requestId }, next);
});

app.post("/checkout/quote", async (req, res, next) => {
  try {
    const context = contextStore.getStore();
    if (!context) throw new Error("Request context is unavailable");

    Object.assign(context, {
      flagKey: "pricing-rule-v2",
      flagVariant: "enabled",
      pricingRuleId: "tiered-discount-17",
    });

    logger.info("Pricing rule evaluated", {
      cart_item_count: Array.isArray(req.body.items) ? req.body.items.length : 0,
    });

    const subtotalCents = 4900;
    const totalCents = 4410;
    logger.info("Quote calculated", { subtotal_cents: subtotalCents, total_cents: totalCents });
    res.json({ total_cents: totalCents, request_id: context.requestId });
  } catch (error) {
    next(error);
  }
});

app.use((error: Error, req: Request, res: Response, _next: NextFunction) => {
  const fields = currentFields({ method: req.method, route: req.route?.path });
  logger.error("Checkout quote failed", error, fields);
  exceptionTracker.captureException(error, {
    tags: {
      "request.id": fields["request.id"],
      "flag.key": fields["flag.key"],
      "flag.variant": fields["flag.variant"],
    },
    extra: {
      "pricing.rule_id": fields["pricing.rule_id"],
      method: fields.method,
      route: fields.route,
    },
  });
  res.status(500).json({
    error: "quote_failed",
    request_id: fields["request.id"],
  });
});

app.listen(3000);
```

There are two details easy to miss. First, `AsyncLocalStorage.run()` establishes the store for callbacks created inside its scope, so the route can read correlation data after an `await` without passing a request object through pricing functions. Second, Express requires asynchronous errors to reach its error pipeline. The explicit `try/catch` and `next(error)` work across commonly deployed Express versions; Express 5 also forwards rejected Promise-returning handlers automatically.

The sample's tracker writes a normalized event to standard error so the example stays vendor-neutral. In production, replace only that object with an initialized error-tracking client. Preserve the tags and extras. Do not replace the logger call: the structured log records the decision trail, while the exception event records the failure and stack.

No customer email, cart contents, authorization header, or card value enters the correlation fields. That restraint is part of correctness. A request ID should locate the trail; it should not become an excuse to copy the request into every event.

## Reconstruct the pricing incident, not merely the stack

Suppose error tracking shows `request.id=67e55044-10b1-426f-9247-bb680e5fe0c8`. Search the log store for that exact value, not a five-second time window. The expected sequence is an inbound request record, `Pricing rule evaluated`, perhaps downstream dependency records, and either `Quote calculated` or `Checkout quote failed`. Filter next by `flag.key=pricing-rule-v2`, `flag.variant=enabled`, and `pricing.rule_id=tiered-discount-17`. Compare that trail with a request from the disabled variant. If the enabled trail reaches rule 17 and stops before `Quote calculated`, while the disabled trail completes, the evidence points toward the pricing decision path; if both stop at the same catalog call, the rollout is probably incidental. Next group captured exceptions by the flag variant and release identifier, then inspect the request-level trails behind the groups rather than treating a spike as proof by itself. The method preserves two views: aggregation shows whether a pattern exists, and the exact request ID reveals the order of decisions inside one failure. Now the team can distinguish a rule-specific pattern from an unrelated checkout failure without copying a customer's cart or identity into telemetry.

That is the payoff.

Keep field names stable across services. OpenTelemetry semantic conventions define `error.type` for identifying an error and standard HTTP attributes for spans; adopting standard names where they fit reduces translation when logs, traces, and exceptions meet later. The custom pricing fields still need a small internal schema. Document their types, owners, and allowed cardinality. In particular, keep a rule identifier or flag key as a tag, but keep free-form error messages and customer identifiers out of indexed tags because their cardinality grows without bound.

Incident reconstruction also needs a deployment marker. Add a release or commit identifier during logger and tracker initialization, not by reading it from each request. Then compare failures across the flag variant and release boundary. This avoids a familiar false inference: a rollout and an application deploy happen close together, and the flag receives blame merely because its value is visible.

Test the contract at three levels. A middleware test should verify that a valid inbound UUID is preserved, an invalid value is replaced, and the response exposes the chosen ID. An adapter test should assert that both logger choices emit the same correlation fields. Finally, an error-route test should use a fake tracker and prove that one thrown exception produces one log call and one capture call with equal `request.id` values. Use exact equality. Timestamp proximity isn't correlation.

## What about duplicate reports, background work, and sampling?

The first objection is duplication. If every layer captures the same thrown error, one failure becomes several exception events. Pick an owner: the outermost Express error middleware captures an unhandled request exception once, while lower layers add useful fields and rethrow. A lower layer may capture a handled error only when it represents a separate operational event; if it does, mark that boundary explicitly. Logger output can contain the error at each meaningful decision point, but repeated stack dumps usually add volume rather than evidence.

The second objection is lost context after the response or inside a queue worker. Request context should not be treated as a permanent global bag. Before enqueueing work, copy only safe correlation fields into the job payload, then establish a fresh `AsyncLocalStorage` scope in the worker. Give the job its own ID and retain the originating request ID as a parent correlation field. This produces a legible chain instead of pretending a five-minute fulfillment task is still an HTTP request.

Sampling needs separate rules for logs and exceptions. Keep exceptions that represent failed requests, but consider sampling high-volume success logs after retaining the pricing decision fields needed for rollout analysis. I'm not sure what rate is appropriate for a given store without its event volume, retention window, query latency, and incident objectives. Those measurements resolve the choice. One fixed percentage copied from another service does not.

## Choose the operational trade-off deliberately

Pino is a sensible fit when low logging overhead and JSON output are the primary constraints; Winston is a sensible fit when an existing service already depends on its formats and transport model. This article does not rank them. The adapter keeps the correlation contract independent of that choice, and migrating the logger does not require editing route-level observability fields.

The catch is that `AsyncLocalStorage` is not suitable as the only propagation mechanism across process boundaries. Use trace context or explicit message metadata when requests cross services and queues. Stick with explicit parameter passing in a small codebase when hidden ambient context would make tests or control flow harder to understand. Also avoid adopting an error tracker merely to store ordinary logs: a log backend and an exception tracker answer different questions, even when a shared request ID lets an operator move between them.

Start the rollout with the flag disabled, exercise both variants in tests, and query by the proposed fields before exposing traffic. The go/no-go check is concrete: given any captured checkout exception, can an on-call engineer retrieve the pricing decision and release using one exact request ID? If the answer is no, adding dashboards will not repair the missing join key. Fix the event contract first.

## References

- https://nodejs.org/api/async_context.html#class-asynclocalstorage
- https://expressjs.com/en/guide/error-handling.html
- https://getpino.io/#/docs/api?id=bindings
- https://github.com/winstonjs/winston#creating-child-loggers
- https://opentelemetry.io/docs/specs/semconv/general/recording-errors/
- https://opentelemetry.io/docs/specs/semconv/http/http-spans/
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
