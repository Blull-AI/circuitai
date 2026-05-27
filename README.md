# @blull/circuitai

> Type-safe, durable, observable orchestration for teams of AI agents.

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
    .enum(["sms_reminder", "outbound_call", "settlement_offer", "legal_escalation"])
    .optional(),
});
type Ctx = z.infer<typeof ContextSchema>;

const lookupPaymentHistory = defineTool({
  name: "lookupPaymentHistory",
  description: "Fetch 90-day payment, promise-to-pay and contact-attempt history for a debtor.",
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
  contextSelector: (ctx: Ctx) => ({ recoverabilityScore: ctx.recoverabilityScore ?? 0 }),
});

const project = defineProject({
  name: "collection-decisioning",
  description: "Score overdue accounts and route each one to the next-best collection action",
  goal: "Produce a next-best collection action for every overdue account in the queue",
  contextSchema: ContextSchema,
  agents: [recoverabilityScorer, actionSelector],
  supervisor: defineSupervisor<Ctx>({
    model: openai,
    rules: "Score the account first, then pick the action. Stop when `nextAction` is set.",
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
2. **Visualizable, serializable graphs.** `project.toJSON()` emits a JSON-Schema description of your agent team. Persist it, diff it in git, render it in a future Studio.
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
```

> Requires Zod **v4** (peer). `openai` and `@anthropic-ai/sdk` are **optional** peers — install only what you use.

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
    case "context.updated":
    case "project.completed":
    case "error":
      break;
  }
}
```

`project.run(initialContext)` is the same loop, but it awaits to completion and returns the final context directly.

## Persistence

```ts
import { createInMemoryStorage } from "@blull/circuitai";

const project = defineProject({
  // ...
  storage: createInMemoryStorage(), // default; swap for your own implementation
});

const finalCtx = await project.run(initial);
const runs = await project.storage.listRuns({ projectName: "collection-decisioning" });
const loaded = await project.storage.loadRun(runs[0]!.id);
console.log(loaded?.events); // full event log
```

Implement the `Storage` interface to plug into Postgres, Redis, S3, or anything else. @blull/circuitai v0.1 ships the in-memory default only; first-party adapters are on the v0.2 roadmap.

## Telemetry

```ts
import { createConsoleTelemetry } from "@blull/circuitai";

const project = defineProject({
  // ...
  telemetry: createConsoleTelemetry({ enabled: true }), // pretty stdout traces
});
```

The `Telemetry` interface mirrors OpenTelemetry's `Tracer`/`Span` shapes so an OTel adapter is a thin shim.

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

Diff your agent team in git, share it across services, render it in a future Studio.

## Testing without API keys

```ts
import { createMockProvider } from "@blull/circuitai/testing";

const agent = defineAgent({
  // ...
  model: createMockProvider({
    responses: [{ structured: { riskScore: 0.72, flags: [] } }],
  }),
});
```

Scripted responses for unit and integration tests. No network. No API keys.

## Examples

- `examples/with-mock-provider/run.ts` — fully offline run using the mock provider.
- `examples/fraud-detection/openai.ts` — same project with OpenAI's `gpt-5.4-mini`.
- `examples/fraud-detection/anthropic.ts` — same project with Anthropic's Claude.

```sh
pnpm tsx examples/with-mock-provider/run.ts
OPENAI_API_KEY=sk-... pnpm tsx examples/fraud-detection/openai.ts
ANTHROPIC_API_KEY=sk-... pnpm tsx examples/fraud-detection/anthropic.ts
```

## Status

@blull/circuitai is **v0.1** — focused on getting orchestration, type safety, persistence, observability, and visualization right. The roadmap:

- **v0.2** — Postgres + Redis storage adapters, OpenTelemetry telemetry adapter, streaming tokens to event consumers.
- **v0.3** — human-in-the-loop pause/resume, durable workflow primitives.
- **v0.4** — @blull/circuitai Studio: a hosted visual editor for the project graph and a live debugger for runs.

## License

MIT
