# Logging — Accordo standard

This is the single source of truth for how we log across all three services
(`Accordo-ai-backend`, `Accordo-auth`, `Accordo-ai-frontend`). The goal is
that someone joining the team six months from now opens a file in any repo
and never has to ask "what logger?", "what level?", or "is this PII?".

> **One-liner:** Pino in both backends, JSON to stdout in prod, redaction at
> the logger config, request IDs propagated between services. Frontend keeps
> a thin `logger.ts` wrapper that strips `debug`/`info` from production
> builds. `console.*` is banned by ESLint; use `logger`.

---

## 1. Levels

Five levels, identical across all services. Pick the lowest level that fits.

| Level     | What it means                                               | Production behavior            | Examples                                                                                                       |
| --------- | ----------------------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| **fatal** | Process is in an unrecoverable state and should exit.       | Pages on-call.                 | Cannot connect to DB after retries. `JWT_ACCESS_TOKEN_SECRET` missing in production.                           |
| **error** | A specific request/operation failed; service keeps running. | Alerts if rate spikes.         | 5xx response. Failed DB write. SMTP send failure. Uncaught exception inside a request.                         |
| **warn**  | Recoverable / suspicious / fallback path taken.             | Visible in dashboards.         | Rate-limit triggered. Validation failed for an authenticated user. Ephemeral JWT secret generated.             |
| **info**  | Routine business event worth keeping.                       | Default prod level. Always on. | Server started. User signed up. Deal created. Request completed (one line per request, with status + latency). |
| **debug** | Useful when investigating a specific issue.                 | **Off in prod by default.**    | "Decision engine picked COUNTER with utility 0.84." LLM prompt body. DB query params (after redaction).        |

**Don't log at `info` for things only a developer cares about** — that's
`debug`. Production `info` should read like a business event log.

---

## 2. Always log with a context object, not a string

```ts
// ❌ Bad — unsearchable, can't group, doesn't survive JSON serialization cleanly
logger.info("user " + userId + " signed up");

// ✅ Good — structured fields aggregators (CloudWatch / Datadog / Loki) can index
logger.info({ userId, event: "user.signup" }, "user signed up");
```

Conventions:

- The **first argument** is the structured context.
- The **second argument** is a short human-readable message — should be a
  stable string, not interpolated. Variables go in the context object.
- Use `event: "domain.verb"` (e.g. `user.signup`, `deal.created`,
  `auth.refresh.failed`) for events you care about querying later.

---

## 3. Required fields (auto-injected by middleware)

Every log line emitted during a request automatically carries:

| Field       | Source                                                     | Always present |
| ----------- | ---------------------------------------------------------- | -------------- |
| `service`   | Logger config (`accordo-backend` / `accordo-auth`)         | yes            |
| `env`       | `NODE_ENV`                                                 | yes            |
| `requestId` | `pino-http` middleware (incoming `x-request-id` or new v7) | per request    |
| `userId`    | Auth middleware (after JWT verified)                       | when authed    |
| `companyId` | Auth middleware                                            | when authed    |

You **don't** add these by hand. The request-scoped child logger does it
for you. From a controller / service, just call `req.log.info({...})` or
import the module logger.

**Cross-service correlation:** when backend calls auth, it forwards
`x-request-id`. Auth's middleware honors it. One `requestId` in your log
aggregator surfaces the whole transaction.

---

## 4. Redaction (enforced at the logger, not by humans)

The logger config redacts these keys **anywhere they appear** in the log
object — top-level, nested, in request bodies, in stringified errors:

- `password`, `newPassword`, `currentPassword`
- `accessToken`, `refreshToken`, `token`
- `apiKey`, `apiSecret`
- `otp`
- `req.headers.authorization`
- `req.headers["x-refresh-token"]`
- `req.headers.cookie`
- `res.headers["set-cookie"]`
- `*.password`, `*.token` (catch-all)

This means **`logger.info({ body: req.body })` is safe by construction**.
Don't try to redact yourself; trust the config. If you find a new sensitive
key, add it to the redaction list once, not at every call site.

---

## 5. Format

- **Dev (`NODE_ENV !== "production"`):** piped through `pino-pretty`,
  colored, human readable.
- **Prod:** raw JSON, one event per line, to stdout. The container
  platform (CloudWatch / Datadog / Loki / Vector) picks it up.

No file logging. No `error.log` / `app.log`. Containers are ephemeral; if
the log doesn't leave the container via stdout, it's lost.

---

## 6. Frontend — same vocabulary, different mechanism

The frontend has no log aggregator (yet). Two rules:

1. **Use `src/utils/logger.ts`**, not `console.*`. Same 5 method names
   (`fatal`, `error`, `warn`, `info`, `debug`).
2. **`debug` / `info` / `warn` are stripped from production builds.**
   Only `error` and `fatal` survive — they go to `console.error` so
   Sentry-like tools can catch them later. The Vite build also drops bare
   `console.*` via `esbuild.drop` as a safety net.

When you wire up Sentry / DataDog RUM, only `error` and `fatal` need to
change to forward to it. One file.

---

## 7. Banned

- **`console.*`** — ESLint `no-console: error`. Use `logger`.
- **Logging full request bodies on auth endpoints** — even with redaction
  on, prefer logging just `{ method, url, userId }` on `/auth/login`,
  `/auth/register`, `/auth/refresh-token`. Defense in depth.
- **Logging the response body of any 2xx** — too noisy, and bodies can
  contain PII the client surfaces but the server shouldn't archive.
- **Multi-line logs.** One event = one line. If you want to attach an
  object, put it in the context, not in the message string.
- **String concatenation in the message.** Use the context object.
- **`logger.info` for things only a developer cares about.** Use `debug`.

---

## 8. Examples by service

### Backend (`Accordo-ai-backend`)

```ts
import logger from "@/config/logger.js";

// Module-scope log (no request context)
logger.info({ port: env.port }, "server started");

// Inside a controller — use req.log so requestId / userId are attached
export const createDeal = async (req, res, next) => {
  try {
    const deal = await dealService.create(req.body, req.context);
    req.log.info({ event: "deal.created", dealId: deal.id }, "deal created");
    res.status(201).json({ data: deal });
  } catch (err) {
    req.log.error(
      { err, event: "deal.create.failed" },
      "failed to create deal",
    );
    next(err);
  }
};
```

### Auth (`Accordo-auth`)

Same shape — auth already uses Pino. Examples from `auth.service.ts`:

```ts
import logger from "@/config/logger.js";

logger.warn(
  { event: "auth.login.failed", email: data.email },
  "login attempt for unknown email",
);
```

### Frontend (`Accordo-ai-frontend`)

```ts
import { logger } from "@/utils/logger";

// Survives prod
logger.error({ err }, "Failed to submit deal");

// Stripped from prod build
logger.debug({ payload }, "Sending deal-create request");
```

---

## 9. Adding a new log line — checklist

Before merging a PR with new log statements, ask:

- [ ] Right level? (Is this really `info`, or is it `debug`?)
- [ ] Context object first, message second? (`logger.X({...}, "msg")`)
- [ ] Variables in the context, not interpolated into the message?
- [ ] No secrets directly named? (If you log `req.body`, redaction will
      catch it — but don't log secrets explicitly.)
- [ ] `event: "domain.verb"` set for anything you'd want to query later?
- [ ] In a controller? Using `req.log` so request/user context attaches?

---

## 10. Operations cheat-sheet

```bash
# Tail backend in prod (JSON)
docker logs -f accordo-backend

# Tail with pretty-printing
docker logs -f accordo-backend | npx pino-pretty

# Find one transaction by request-id
docker logs accordo-backend accordo-auth 2>&1 | grep '"requestId":"8c4f9a3e"'

# Raise log level dynamically (only at boot — needs container restart)
LOG_LEVEL=debug docker compose --profile prod up -d backend
```

---

## 11. References

- Pino docs: https://getpino.io/
- Pino redaction: https://getpino.io/#/docs/redaction
- `pino-http` (request middleware + requestId): https://github.com/pinojs/pino-http
- `pino-pretty` (dev formatter): https://github.com/pinojs/pino-pretty

For Pino config in this repo:

- Backend: `Accordo-ai-backend/src/config/logger.ts`
- Auth: `Accordo-auth/src/config/logger.ts`
- Frontend: `Accordo-ai-frontend/src/utils/logger.ts`
