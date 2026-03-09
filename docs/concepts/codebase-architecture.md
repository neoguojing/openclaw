---
summary: "Repository-level architecture map for OpenClaw core, gateway, channels, and extensions"
read_when:
  - Getting started with OpenClaw code navigation
  - Planning cross-cutting refactors across gateway, channels, and agents
  - Reviewing ownership boundaries between core, plugins, and extensions
title: "Codebase Architecture"
---

# Codebase architecture

Last updated: 2026-03-09

## 1) System shape at a glance

OpenClaw is organized as a **single runtime core** (`src/`) with three major integration surfaces:

- **Gateway/control plane**: WebSocket + HTTP server lifecycle, auth, pairing, subscriptions, and server methods.
- **Channel plane**: inbound/outbound messaging adapters, routing/session binding, and per-channel onboarding/config.
- **Agent/tool plane**: agent execution, tools, memory/context, and multi-agent runtime scaffolding.

The runtime is bootstrapped through `openclaw.mjs` and `src/index.ts`, then composed by the CLI program builder and command registry.

## 2) Bootstrap and entrypoints

### Runtime bootstrap

- `openclaw.mjs` enforces Node baseline (`>=22.12`), enables compile cache where available, installs warning filters, then loads build output (`dist/entry.*`).
- `src/index.ts` performs runtime guards (`dotenv`, env normalization, PATH setup, runtime checks, log capture), builds the Commander program, and handles top-level process errors.

### CLI composition

- `src/cli/program.ts` delegates to `buildProgram`.
- `src/cli/program/build-program.ts` wires context creation, help/pre-action hooks, and command registration in one place.

This gives a clear split between:

1. **bootstrap safety** (runtime checks, env guards),
2. **command graph composition** (Commander wiring), and
3. **command implementation modules** under `src/commands/*`.

## 3) Source tree architecture (core)

`src/` is large but follows stable domain groupings:

- **Gateway and protocol**
  - `src/gateway/`: server runtime, WS methods, auth, health, reload, tailscale exposure, node integration.
  - `src/gateway/protocol/`: protocol contracts and transport-facing schema helpers.
- **CLI and commands**
  - `src/cli/`: program wiring and command registration.
  - `src/commands/`: user-facing command implementations grouped by feature.
- **Channels and routing**
  - `src/channels/`: channel metadata/registry, shared policies, plugin loading bridges.
  - `src/routing/`: account lookup, route resolution, session key derivation and continuity.
  - Channel families under `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/line`, `src/web`.
- **Agent runtime and cognition**
  - `src/agents/`: core agent loop, tooling integration, subagent/runtime boundaries, schemas.
  - `src/context-engine/`, `src/memory/`, `src/media-understanding/`, `src/link-understanding/` for context/media augmentation.
- **Plugin and extension contract**
  - `src/plugins/`: runtime plugin registry, services, lifecycle glue.
  - `src/plugin-sdk/`: public SDK surface exported to extension packages.
- **Shared infrastructure**
  - `src/infra/`, `src/process/`, `src/security/`, `src/secrets/`, `src/logging/`, `src/config/`.

## 4) Plugin and channel boundaries

OpenClaw keeps channel normalization lightweight in `src/channels/registry.ts` and intentionally avoids eager heavy channel imports there. Runtime channel/plugin resolution comes from the active plugin registry in `src/plugins/runtime.ts`.

This enables:

- low-cost channel ID normalization in shared code paths,
- lazy(er) channel implementation loading,
- support for both bundled channels and external extension channels.

## 5) Gateway as orchestration nucleus

`src/gateway/server.impl.ts` acts as the integration hub and composes dependencies from many domains:

- config loading/migration,
- channel manager startup,
- agent event handling,
- plugin runtime activation,
- secrets runtime snapshot,
- health/heartbeat/cron/discovery sidecars,
- WS method registration and node subscription managers.

The file is intentionally orchestration-heavy: feature logic should remain in focused submodules that it imports, while startup and lifecycle ordering stay centralized.

## 6) Workspace-level architecture outside `src/`

- `extensions/*`: workspace packages for optional channels/features. Runtime plugin dependencies should remain extension-local.
- `apps/*`: platform apps (macOS/iOS/Android) and related host-side integration.
- `docs/*`: Mintlify docs; product architecture docs live under `docs/concepts/`.
- `scripts/*`: operational, release, test, and packaging automation.

## 7) Recommended dependency direction

When adding new features, keep dependencies flowing inward toward stable abstractions:

1. `cli/commands` → use service/domain modules, avoid deep imports into unrelated adapters.
2. `gateway` → orchestrates domain modules; avoid embedding feature logic directly in top-level server wiring.
3. `channels` → use shared routing/session policy modules, keep channel-specific behavior isolated.
4. `plugin-sdk` → versioned contract; avoid leaking internal-only runtime types.

## 8) Refactor checklist (practical)

For cross-cutting changes, verify all layers below before landing:

- protocol/schema compatibility (`gateway/protocol`, client impact),
- channel registry + aliases + onboarding coverage,
- routing/session continuity (`src/routing/*`),
- extension compatibility (`extensions/*`),
- CLI surface and docs synchronization (`src/cli`, `src/commands`, `docs/`).
