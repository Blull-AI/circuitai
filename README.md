<p align="center">
  <a href="https://sdk.blull.com.br">
    <img src="https://raw.githubusercontent.com/Blull-AI/circuitai/main/assets/logo.png" alt="Blull" width="260" />
  </a>
</p>

# @blull/circuitai

> Type-safe, durable, observable orchestration for teams of AI agents.

**Official website:** [sdk.blull.com.br](https://sdk.blull.com.br)

@blull/circuitai helps you orchestrate teams of AI agents the way you'd orchestrate a real project team. Each agent has a role, a goal, and rules. A supervisor coordinates them toward a business goal. Everything is typed end-to-end with Zod — the context schema, every agent's input and output, every tool's input — and every run is a stream of structured events you can inspect, persist, and replay.

```ts
import { z } from "zod";
import {
  createOpenAIProvider,
  defineAgent,
  defineProject,
  defineSupervisor,
  defineTool,
} from "@blull/circuitai";

const openai = createOpenAIProvider({
  apiKey: process.env.OPENAI_API_KEY!,
  model: "gpt-5.4-mini",
});

const ContextSchema = z.object({
  account: z.object({
    id: z.string(),
    debtorName: z.string(),
    outstandingUsd: z.number(),
    daysPastDue: z.number(),
    productType: z.enum(["credit_card", "personal_loan", "utility", "telecom"]),
  }),
  recoverabilityScore: z.number().min(0).max(1).optional(),
  nextAction: z
    .enum([
      "sms_reminder",
      "outbound_call",
      "settlement_offer",
      "legal_escalation",
    ])
    .optional(),
});
type Ctx = z.infer<typeof ContextSchema>;

const lookupPaymentHistory = defineTool({
  name: "lookupPaymentHistory",
  description:
    "Fetch 90-day payment, promise-to-pay and contact-attempt history for a debtor.",
  schema: z.object({ accountId: z.string() }),
  handler: async ({ accountId }) => ({
    accountId,
    partialPayments: 1,
    brokenPromises: 2,
    callAttempts: 5,
    rightPartyContacts: 1,
  }),
});

const recoverabilityScorer = defineAgent({
  name: "recoverability-scorer",
  goal: "Estimate how likely this overdue account is to be collected",
  rules: "You are a collections analyst at a debt-recovery agency. JSON only.",
  model: openai,
  inputSchema: z.object({ account: ContextSchema.shape.account }),
  outputSchema: z.object({ recoverabilityScore: z.number().min(0).max(1) }),
  contextSelector: (ctx: Ctx) => ({ account: ctx.account }),
  tools: [lookupPaymentHistory],
});

const actionSelector = defineAgent({
  name: "action-selector",
  goal: "Pick the next-best collection action for the call center to dispatch",
  rules:
    "Map the recoverability score to one action: low → legal_escalation, mid → settlement_offer, high → outbound_call, very high → sms_reminder. JSON only.",
  model: openai,
  inputSchema: z.object({ recoverabilityScore: z.number() }),
  outputSchema: z.object({
    nextAction: z.enum([
      "sms_reminder",
      "outbound_call",
      "settlement_offer",
      "legal_escalation",
    ]),
  }),
  contextSelector: (ctx: Ctx) => ({
    recoverabilityScore: ctx.recoverabilityScore ?? 0,
  }),
});

const project = defineProject({
  name: "collection-decisioning",
  description:
    "Score overdue accounts and route each one to the next-best collection action",
  goal: "Produce a next-best collection action for every overdue account in the queue",
  contextSchema: ContextSchema,
  agents: [recoverabilityScorer, actionSelector],
  supervisor: defineSupervisor<Ctx>({
    model: openai,
    rules:
      "Score the account first, then pick the action. Stop when `nextAction` is set.",
    terminationCondition: (ctx) => ctx.nextAction !== undefined,
    maxTurns: 8,
  }),
});

const finalContext = await project.run({
  account: {
    id: "acc_1042",
    debtorName: "John Smith",
    outstandingUsd: 4_200,
    daysPastDue: 47,
    productType: "credit_card",
  },
});
console.log(finalContext.nextAction);
// 'sms_reminder' | 'outbound_call' | 'settlement_offer' | 'legal_escalation'
```

## Why @blull/circuitai

1. **End-to-end typed with Zod.** Your context schema, every agent's input/output, every tool's input — one Zod schema each. Inference flows from `defineProject` all the way to `project.run(initialContext)`. No `as any`. No DSL.
2. **Visualizable, serializable graphs.** `project.toJSON()` emits a JSON-Schema description of your agent team. Persist it, diff it in git, or open it in **Studio** (`@blull/circuitai/studio`) — a self-hostable graph viewer and live run debugger.
3. **Built-in durability.** Every run is a sequence of typed events against a typed context. The in-memory default ships out of the box; implement one `Storage` adapter to plug into Postgres, Redis, or anything else.
4. **Hierarchical orchestration, not graph wiring.** No DAGs to draw. A supervisor LLM routes work dynamically based on the typed shared context. Bring your routing rules in plain English.
5. **First-class tools, first-class observability.** Define tools with Zod schemas; @blull/circuitai runs the model → tool → model loop with validation and retries. Every step is a structured event you can stream, log, or replay.

## Concepts

- **Project** — name, description, goal, the Zod-typed shared context (the _blackboard_), agents, supervisor, optional tasks.
- **Agent** — name, goal, rules (system prompt), an LLM provider, an input/output Zod schema, a `contextSelector` that picks the slice of context this agent needs, and optional tools.
- **Supervisor** — special agent with privileges. Reads the full context, decides the next action (`invoke` an agent, run a `task`, mark `done`, or `fail`). Powered by its own LLM.
- **Task** _(optional)_ — a named, ordered list of agent steps with conditional `when` filters. The supervisor may choose to delegate to a task instead of orchestrating step-by-step.
- **Tool** — a typed function the agent may call during its turn (`schema` Zod + `handler`).

The supervisor's decision is enforced by a Zod-validated discriminated union; invalid choices (hallucinated agent names, bad JSON) trigger bounded corrective retries before the run aborts.

## Installation

```sh
pnpm add @blull/circuitai zod
# plus the providers you want:
pnpm add openai @anthropic-ai/sdk
# plus any durable adapters you want:
pnpm add pg                  # Postgres storage
pnpm add ioredis             # Redis storage
pnpm add @opentelemetry/api  # OpenTelemetry traces
```

> Requires Zod **v4** (peer). `openai`, `@anthropic-ai/sdk`, `pg`, `ioredis`, and `@opentelemetry/api` are all **optional** peers — install only what you use.

## Run lifecycle and events

Every run emits a typed event stream. Stream them for live UI, persist them for audit, or replay them for debugging:

```ts
for await (const event of project.stream(initialContext)) {
  switch (event.type) {
    case "project.started":
      break;
    case "supervisor.decision":
    case "agent.started":
    case "agent.tool_called":
    case "agent.completed":
    case "agent.delta": // streamed content tokens (opt-in, see below)
    case "context.updated":
    case "project.paused": // suspended for human input (see below)
    case "project.resumed":
    case "project.completed":
    case "error":
      break;
  }
}
```

`project.run(initialContext)` is the same loop, but it awaits to completion and returns the final context directly.

### Streaming tokens

Opt in with `{ stream: true }` to receive token-level `agent.delta` events as each agent's model produces them — ideal for live UIs:

```ts
for await (const event of project.stream(initialContext, { stream: true })) {
  if (event.type === "agent.delta") {
    process.stdout.write(event.delta); // live token-by-token output
  }
}
```

Deltas require the agent's provider to implement streaming (the OpenAI and Anthropic adapters do); providers without it fall back to a single non-streaming call. Deltas are **ephemeral** — they reach live consumers but are never written to storage, since the durable `agent.completed` event already carries the agent's complete output. Without `{ stream: true }` the stream behaves exactly as before.

## Persistence

```ts
import { createInMemoryStorage } from "@blull/circuitai";

const project = defineProject({
  // ...
  storage: createInMemoryStorage(), // default; swap for your own implementation
});

const finalCtx = await project.run(initial);
const runs = await project.storage.listRuns({
  projectName: "collection-decisioning",
});
const loaded = await project.storage.loadRun(runs[0]!.id);
console.log(loaded?.events); // full event log
```

First-party durable adapters ship for Postgres and Redis:

```ts
import { createPostgresStorage, createRedisStorage } from "@blull/circuitai";

// Postgres (requires `pg`)
const pg = createPostgresStorage({
  connectionString: process.env.DATABASE_URL,
});
await pg.ensureSchema(); // create the runs/events tables once — or run pg.schemaSql yourself

// Redis (requires `ioredis`)
const redis = createRedisStorage({
  url: process.env.REDIS_URL,
  ttlSeconds: 60 * 60 * 24,
});

const project = defineProject({
  // ...
  storage: pg, // or redis
});
```

Both implement the same `Storage` interface. Pass a pre-built `pool` / `client` instead of a URL when you want full connection control (and trivial test injection). Streamed `agent.delta` events are intentionally **not** persisted. You can still implement `Storage` yourself for any other backend (S3, DynamoDB, …).

## Human-in-the-loop: pause & resume

A run can **suspend itself for human input** and be **resumed later** — even from a different process — without re-running any completed work. The `project.paused` event written to storage _is_ the durable checkpoint: it carries the full blackboard context, the turn cursor, and accumulated usage. `project.resume(runId)` restores that state and continues the supervisor loop.

There are two ways to pause:

1. **A deterministic gate** — `supervisor.pauseCondition`, checked each turn _before_ the supervisor LLM is consulted (so a gate pause costs zero supervisor calls). This is the robust primitive for approval flows.
2. **A supervisor decision** — the supervisor LLM can choose `{ kind: 'pause', reason, awaiting }` when it decides it needs a human.

```ts
const project = defineProject({
  // ...
  contextSchema: z.object({
    amount: z.number(),
    prepared: z.boolean().optional(),
    approval: z.enum(["approved", "rejected"]).optional(),
    executed: z.boolean().optional(),
  }),
  agents: [preparer, executor],
  supervisor: defineSupervisor({
    model: openai,
    rules: "Prepare the action, then execute it only once it is approved.",
    terminationCondition: (ctx) => ctx.executed === true,
    // Pause once the action is prepared but not yet approved. The gate MUST be
    // resolved by the resume input — otherwise the resumed run pauses again.
    pauseCondition: (ctx) =>
      ctx.prepared && !ctx.approval
        ? { reason: "approval required", awaiting: "approve | reject" }
        : undefined,
  }),
});

// Run until it pauses. `run()` throws `run_paused` (it has no final context to
// return); use `stream()` to observe the pause as an event instead.
let runId = "";
for await (const event of project.stream(initialContext)) {
  if (event.type === "project.paused") {
    runId = event.runId;
    console.log(event.reason, event.awaiting); // surface to your reviewer UI
  }
}

// ...later, possibly in another process — only the runId + shared storage are
// needed. `resume` returns the same event stream as `stream()`; the human's
// decision is merged into the restored context.
let final;
for await (const event of project.resume(runId, {
  input: { approval: "approved" },
})) {
  if (event.type === "project.completed") final = event.context;
}
```

Resume continues from the checkpoint: the already-completed `preparer` is **not** re-invoked; the supervisor picks up from the restored context and runs `executor`. The full lifecycle (`project.started → … → project.paused → project.resumed → … → project.completed`) lives in one event log for audit.

**Requirements and limits**

- Resume needs the **same project definition** (agents, supervisor, tools) and the **same `Storage`** the run was created with. The code is the workflow; storage is the durable state.
- The restored context must be **JSON-round-trippable** through your storage backend. `z.date()`, `Map`, `Set`, etc. drift through JSON; resume re-validates the restored context against your schema and fails fast at the `resume` stage if it does. Prefer JSON-native context (ISO strings, plain objects).
- A run is resumable only while **paused** (`resume_not_found` / `resume_not_resumable` otherwise). The resume `input` is merged then re-validated; an invalid merge throws `schema_validation` and leaves the run cleanly paused.
- **No resume locking (v0.3).** Two concurrent `resume(runId)` calls both load the same checkpoint and proceed — there is no atomic compare-and-swap on status. Serialize resumes of the same run yourself.

## Telemetry

```ts
import { createConsoleTelemetry } from "@blull/circuitai";

const project = defineProject({
  // ...
  telemetry: createConsoleTelemetry({ enabled: true }), // pretty stdout traces
});
```

The `Telemetry` interface mirrors OpenTelemetry's `Tracer`/`Span` shapes. To export to a real OTel pipeline, hand `createOtelTelemetry` a `Tracer` from your own SDK setup:

```ts
import { createOtelTelemetry } from "@blull/circuitai";
import { trace } from "@opentelemetry/api";

const project = defineProject({
  // ...
  telemetry: createOtelTelemetry({ tracer: trace.getTracer("my-app") }),
});
```

The adapter imports **only types** from `@opentelemetry/api`, so the library never pulls in the OTel SDK at runtime — you own sampling, batching, and export.

## Visualizable graph

```ts
const graph = project.toJSON();
// {
//   schemaVersion: '1.0',
//   name: 'collection-decisioning',
//   agents: [{ name, goal, rules, model, inputSchema: <JSONSchema>, outputSchema: <JSONSchema>, tools: [...] }],
//   supervisor: { model, rules, maxTurns },
//   tasks: [...],
//   contextSchema: <JSONSchema>,
// }
```

Diff your agent team in git, share it across services, or open it in **Studio** (next section).

## Studio

`@blull/circuitai/studio` is a self-hostable web surface that **visualizes your project graph** and **debugs runs** — replaying a finished run from storage, live-tailing one in flight, and resuming paused runs (HITL) from the browser. It reads everything from the `Storage` your projects already write to, plus `project.toJSON()` for the graph, so there is nothing new to instrument.

```ts
import { createStudioServer } from "@blull/circuitai/studio";

// `storage` is the SAME instance your projects use, so Studio sees their runs.
const studio = createStudioServer({
  storage,
  projects: [project], // optional — enables the graph view + HITL resume
});
studio.listen(3030); // http://localhost:3030
```

`createStudioServer` returns a framework-agnostic Node request `handler` (mount it in Express/etc.) plus a `listen()` convenience — **zero new runtime dependencies** (raw `node:http`). It exposes a small read API over your storage (`GET /api/projects`, `/api/projects/:name/graph`, `/api/runs`, `/api/runs/:id`), live events over SSE (`GET /api/runs/:id/events` — replays a finished run, tails a running one), and `POST /api/runs/:id/resume` for paused runs. Mount it under a sub-path with `basePath`.

Two views:

- **Graph** — the supervisor → agents → tools graph rendered from `toJSON()`, with each node's context/input/output JSON Schemas inspectable and the **active agent/tool node highlighted as a run streams**. "Export JSON" downloads the `ProjectGraph`.
- **Timeline** — the full event log with a **step scrubber**: scrub any run to see the reconstructed context snapshot and the per-step diff, token/cost totals, and — for paused runs — a resume box that merges your input and continues the run.

Studio is deliberately read-only over your data: it **does not launch runs** (that needs your providers/keys) and ships **no auth** — put it behind your own gateway if you expose it. Pass live `Project` instances to enable the graph view and resume; omit them for a pure run-debugger over any `Storage` backend (e.g. a shared Postgres). The graph view visualizes and exports the `ProjectGraph`; regenerating runnable code from an edited graph (`fromJSON`/codegen) is not part of v0.4.

Try it offline — no API key (mock provider, with seeded and live runs):

```sh
pnpm tsx examples/studio/run.ts   # http://localhost:3030
```

## Testing without API keys

```ts
import { createMockProvider } from "@blull/circuitai/testing";

const agent = defineAgent({
  // ...
  model: createMockProvider({
    responses: [{ structured: { recoverabilityScore: 0.72, flags: [] } }],
  }),
});
```

Scripted responses for unit and integration tests. No network. No API keys.

## Examples

- `examples/with-mock-provider/run.ts` — fully offline run using the mock provider.
- `examples/human-in-the-loop/run.ts` — fully offline pause → human approval → resume, where a _second_ project instance resumes the run from shared storage (mimicking another process).
- `examples/studio/run.ts` — fully offline Studio: seeds completed/paused runs and a live-run generator into a shared in-memory store, then serves Studio at `http://localhost:3030`.
- `examples/cobranca/openai.ts` — same project with OpenAI's `gpt-5.4-mini`.
- `examples/cobranca/anthropic.ts` — same project with Anthropic's Claude.

```sh
pnpm tsx examples/with-mock-provider/run.ts
pnpm tsx examples/human-in-the-loop/run.ts
pnpm tsx examples/studio/run.ts          # then open http://localhost:3030
OPENAI_API_KEY=sk-... pnpm tsx examples/cobranca/openai.ts
ANTHROPIC_API_KEY=sk-... pnpm tsx examples/cobranca/anthropic.ts
```

## License

MIT
