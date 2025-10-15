# Enterprise module reimplementation checklist

The proprietary **gatekeeper** and **workspaces** stacks ship with tightly coupled runtime modules, APIs, persistence logic, and background jobs. To recreate their behaviour without reusing Speckle's licensed code, reimplement each responsibility below while preserving observable logic and integrations.

## Gatekeeper runtime (`packages/server/modules/gatekeeper`)

- Module bootstrap that registers scopes, validates enterprise licences, mounts billing routes, wires Stripe-aware cron jobs, and enforces read-only plan guards for core object/version creation hooks.【F:packages/server/modules/gatekeeper/index.ts†L1-L156】
- Billing REST surface that accepts Stripe webhooks, reconciles checkout sessions, provisions workspace plans/subscriptions, and logs outcomes for async payment flows.【F:packages/server/modules/gatekeeper/rest/billing.ts†L1-L188】
- GraphQL resolvers that expose workspace plan metadata, subscription state, billing portal links, feature gates, seat assignments, and checkout orchestration; includes dataloaders and feature-flag checks.【F:packages/server/modules/gatekeeper/graph/resolvers/index.ts†L1-L200】
- Stripe client utilities that create billing portal sessions, read/update subscription state, list recurring prices, and reconcile subscription items with billing proration semantics.【F:packages/server/modules/gatekeeper/clients/stripe.ts†L1-L167】
- Licence and feature authorisation services that validate signed entitlement tokens, derive allowed module combinations, and gate premium workspace features such as SSO or regional data residency.【F:packages/server/modules/gatekeeper/services/validateLicense.ts†L1-L76】【F:packages/server/modules/gatekeeper/services/featureAuthorization.ts†L1-L45】
- Persistence layer for workspace plans, subscriptions, checkout sessions, and seat allocations, including helpers that join workspace tables, upsert paid/unpaid plan variants, and count seat usage by type.【F:packages/server/modules/gatekeeper/repositories/billing.ts†L1-L120】【F:packages/server/modules/gatekeeper/repositories/workspaceSeat.ts†L1-L120】
- Subscription management services that downscale seats, upgrade plans, emit plan events, and integrate with background schedulers—tied into cron tasks seeded during module init.【F:packages/server/modules/gatekeeper/index.ts†L45-L144】

## Gatekeeper core (`packages/server/modules/gatekeeperCore`)

- Shared domain contracts for plan pricing, billing events, and GraphQL return types that other modules consume to stay type-safe.【F:packages/server/modules/gatekeeperCore/domain/billing.ts†L1-L20】【F:packages/server/modules/gatekeeperCore/domain/events.ts†L1-L38】【F:packages/server/modules/gatekeeperCore/helpers/graphTypes.ts†L1-L8】
- Database migrations underpinning enterprise billing tables (plans, subscriptions, checkout sessions, seat metadata) that must be recreated to maintain schema compatibility.【24c8cf†L1-L8】

## Workspaces runtime (`packages/server/modules/workspaces`)

- Module bootstrap that enforces licence checks, registers workspace scopes/roles, schedules cleanup & tracking jobs, and initialises event listeners spanning multi-region deployments.【F:packages/server/modules/workspaces/index.ts†L1-L148】
- GraphQL resolvers covering workspace CRUD, invites, domain governance, project membership, seat assignments, discoverability, and multi-region replication logic for projects and streams.【F:packages/server/modules/workspaces/graph/resolvers/workspaces.ts†L1-L120】
- Event listeners that respond to invite lifecycle changes, project moves, seat updates, and billing events, updating ACLs, Mixpanel tracking, and Stream permissions accordingly.【F:packages/server/modules/workspaces/events/eventListener.ts†L1-L200】
- REST SSO router implementing OIDC provider registration, validation workflows, stateful auth middleware, invite handling, and seat provisioning while honouring feature gates.【F:packages/server/modules/workspaces/rest/sso.ts†L1-L200】
- Workspace seat services ensuring default seat selection, validation against roles, seat assignment/upserts, and event emission for seat lifecycle changes.【F:packages/server/modules/workspaces/services/workspaceSeat.ts†L1-L120】
- Ancillary services for workspace management, invites, domains, tracking, and project-role mapping referenced throughout resolvers and event handlers.【F:packages/server/modules/workspaces/graph/resolvers/workspaces.ts†L1-L120】【F:packages/server/modules/workspaces/events/eventListener.ts†L1-L140】

## Workspaces core (`packages/server/modules/workspacesCore`)

- Domain types, constants, and event payloads representing workspaces, domains, ACLs, seat types, join requests, and associated events consumed by higher-level modules.【F:packages/server/modules/workspacesCore/domain/types.ts†L1-L101】【F:packages/server/modules/workspacesCore/domain/constants.ts†L1-L6】【F:packages/server/modules/workspacesCore/domain/events.ts†L1-L60】
- Domain operations used by shared services (e.g., retrieving roles and seat state for multiple users) that must be mirrored for API compatibility.【F:packages/server/modules/workspacesCore/domain/operations.ts†L1-L33】
- Database migrations defining workspace tables, domains, ACL timestamps, SSO metadata, join requests, feature flags, and seat defaults that underpin enterprise workspace behaviour.【941d69†L1-L15】

Recreating these elements with clean-room code will let you match Speckle Enterprise behaviour without reusing their protected implementations.
