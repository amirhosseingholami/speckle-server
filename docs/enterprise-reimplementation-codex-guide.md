# Clean-room enterprise module tasks for Codex agents

This guide converts the high-level responsibilities in
[`docs/enterprise-reimplementation-checklist.md`](./enterprise-reimplementation-checklist.md)
into actionable tickets and provides a prompt template for delegating the work to
Codex agents while preserving the clean-room separation.

## Working assumptions

- The agent must **not** open, copy, or refer to the proprietary enterprise
  source files listed in the checklist. Share only behavioural requirements and
  public contracts (GraphQL schemas, REST shapes, database schema targets).
- Every task should result in independent, reviewable commits that compile or
  at least type-check within the OSS server.
- Prefer stubbing integrations (Stripe, OIDC, telemetry) behind interfaces so
  they can be swapped for production adapters later.

## Task queue

Execute the tasks sequentially. If a task produces large diffs, split it into
smaller subtasks that keep tests runnable.

### 1. Gatekeeper clean-room foundation

1. Scaffold a new package (e.g. `packages/server/modules/enterprise-gatekeeper`)
   with module registration hooks, dependency injection container, and feature
   flags mirroring the public API of the proprietary gatekeeper module.
2. Define TypeScript interfaces for licence validation, plan evaluation, and
   seat accounting based on the OSS server contracts that currently depend on
   `gatekeeperCore`.
3. Implement in-memory adapters (no external APIs) for licence checks and plan
   storage, backed by new persistence repositories using Prisma schema drafts.
4. Port GraphQL schema definitions (types, queries, mutations) needed by the
   OSS server and create resolvers that call the new interfaces.
5. Expose REST endpoints for billing webhooks and checkout orchestration with
   placeholder handlers that log requests and return deterministic responses for
   integration tests.

### 2. Shared gatekeeper core contracts

1. Create a `packages/server/modules/enterprise-gatekeeper-core` workspace that
   houses domain types (plans, seats, billing events) in pure TypeScript.
2. Reproduce only the shapes required by OSS packages (consult usage sites via
   search) and write unit tests to guarantee stability.
3. Draft Prisma migrations for plan, subscription, and seat tables; keep them
   idempotent and guarded behind a feature flag.

### 3. Workspaces runtime replacement

1. Add a `packages/server/modules/enterprise-workspaces` module that registers
   workspace routes, hooks, and background jobs without touching the original
   enterprise module.
2. Recreate GraphQL resolvers for workspace CRUD, invite handling, and seat
   assignment using the new gatekeeper contracts where necessary.
3. Implement REST/OIDC flows with configuration-driven providers; start with a
   memory-backed state store and leave TODOs for production secrets.
4. Wire event listeners for invites, seat changes, and project moves—emit
   telemetry events through the existing OSS event bus.

### 4. Workspaces core domain layer

1. Publish domain constants, types, and helper functions inside
   `packages/server/modules/enterprise-workspaces-core` to satisfy OSS
   dependencies.
2. Supply database migrations for workspace metadata (domains, SSO config,
   feature flags) designed from the public schema expectations referenced in the
   checklist.
3. Add integration tests validating that the new domain layer interacts
   correctly with the runtime replacement.

### 5. Replacement hardening & integration

1. Replace placeholder adapters (Stripe, Mixpanel, email) with configuration
   that allows mocking in OSS tests and swapping to production implementations.
2. Ensure migrations run in CI by updating the Prisma generation scripts and
   adding regression tests for critical GraphQL flows.
3. Document how to enable the clean-room modules and how they differ from the
   enterprise originals so downstream consumers can adopt them safely.

## Prompt template for Codex agents

Use the following scaffold when engaging Codex agents for each task. Fill in the
bracketed sections with task-specific details.

```
You are contributing to the Speckle OSS server. Work in a clean-room branch that
must not reference proprietary enterprise code. Implement the next task in the
"Clean-room enterprise module tasks for Codex agents" guide.

Task focus: [paste the numbered task description].

Acceptance criteria:
- [list behavioural outcomes, e.g. "GraphQL query X returns plan data from the
  new repository"]
- [include test requirements, e.g. "unit tests covering Y" or "update existing
  fixtures"]

Constraints:
- Do not open or copy code from `packages/server/modules/gatekeeper*` or
  `packages/server/modules/workspaces*`.
- Keep changes self-contained and runnable with `yarn test server`.

When you finish, summarise the diff and the tests you ran.
```

## Verifying Codex output

- Review the diff manually to ensure no enterprise paths were touched.
- Run the relevant unit/integration tests locally.
- Require the agent to produce or update documentation describing the behaviour
  it implemented.

Following this workflow will let multiple Codex agents tackle the clean-room
reimplementation safely while staying aligned with the high-level checklist.
