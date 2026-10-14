# AGENTS.md

## Task Completion Requirements

- All of `bun fmt`, `bun lint`, and `bun typecheck` must pass before considering tasks completed.
- NEVER run `bun test`. Always use `bun run test` (runs Vitest).

## Project Snapshot

T3 Code is a minimal web GUI for using coding agents like Codex and Claude.

This repository is a VERY EARLY WIP. Proposing sweeping changes that improve long-term maintainability is encouraged.

## Core Priorities

1. Performance first.
2. Reliability first.
3. Keep behavior predictable under load and during failures (session restarts, reconnects, partial streams).

If a tradeoff is required, choose correctness and robustness over short-term convenience.

## Maintainability

Long term maintainability is a core priority. If you add new functionality, first check if there is shared logic that can be extracted to a separate module. Duplicate logic across multiple files is a code smell and should be avoided. Don't be afraid to change existing code. Don't take shortcuts by just adding local logic to solve a problem.

## Package Roles

- `apps/server`: Node.js WebSocket server. Wraps Codex app-server (JSON-RPC over stdio), serves the React web app, and manages provider sessions.
- `apps/web`: React/Vite UI. Owns session UX, conversation/event rendering, and client-side state. Connects to the server via WebSocket.
- `packages/contracts`: Shared effect/Schema schemas and TypeScript contracts for provider events, WebSocket protocol, and model/session types. Keep this package schema-only — no runtime logic.
- `packages/shared`: Shared runtime utilities consumed by both server and web. Uses explicit subpath exports (e.g. `@t3tools/shared/git`) — no barrel index.

## Codex App Server (Important)

T3 Code is currently Codex-first. The server starts `codex app-server` (JSON-RPC over stdio) per provider session, then streams structured events to the browser through WebSocket push messages.

How we use it in this codebase:

- Session startup/resume and turn lifecycle are brokered in `apps/server/src/codexAppServerManager.ts`.
- Provider dispatch and thread event logging are coordinated in `apps/server/src/providerManager.ts`.
- WebSocket server routes NativeApi methods in `apps/server/src/wsServer.ts`.
- Web app consumes orchestration domain events via WebSocket push on channel `orchestration.domainEvent` (provider runtime activity is projected into orchestration events server-side).

Docs:

- Codex App Server docs: https://developers.openai.com/codex/sdk/#app-server

## Reference Repos

- Open-source Codex repo: https://github.com/openai/codex
- Codex-Monitor (Tauri, feature-complete, strong reference implementation): https://github.com/Dimillian/CodexMonitor

Use these as implementation references when designing protocol handling, UX flows, and operational safeguards.

Provenance: https://github.com/namelesstherebel/saddle/blob/45fe8c4aa2b3deb644c5baa18980c80c67848b85/templates/AGENTS.md

Adoption snapshot base_sha40: `b8305afa29309e52045987caab91db9b7e481ac0`. Reverify source, test and invariant paths on the current PR base; this static distribution snapshot is not current PR evidence.

## Code Review Rules
Adopter guidance only; it needs no YAML. Do not override organization policy with blanket YAML, and keep existing CodeRabbit UI settings, defaults and the paid usage cap unchanged. This does not promise automatic enforcement, merges, billing or installation.

Reviewer roles:

- CodeRabbit: localized runtime review of the diff.
- Native Codex: cross-module review. Model is unknown; no guaranteed Sol.
- Semgrep: known static patterns.
- Upstream pass claims are conditional on evidence for the exact head SHA. Dedupe root causes, retain the new impact. Reasoning is not execution.

Context to cite, never invent: base SHA the current PR base SHA, obtained from GitHub for this review and cited explicitly (not the inventory snapshot SHA); invariant docs `AGENTS.md`, `CLAUDE.md`, `README.md`. Source paths: `apps/**`, `packages/**`, `scripts/**`. Test paths: `**/test/**`, `**/*.test.*`.

### Reviewer prompts

```text
Code path instructions:
You are our front-line runtime-simulation gate. Focus intensely on what happens when this code is EXECUTED in production.
- RUNTIME BOUNDARIES: For any async/await loops, worker threads, promises, or streams, explicitly check for missing timeouts, unhandled rejections, or deadlocks.
- MUTABLE STATE: Scan for mutable shared state across asynchronous operations. Flag if a race condition is possible under concurrent load.
- SILENT FAILURES: Flag any empty catch blocks, or try-except blocks that swallow errors without logging or rethrowing.

Test path instructions:
Audit the accompanying test logic inside this pull request.
- STRATEGY VALIDATION: Verify tests cover edge-cases: null/empty payloads, network drop simulation, and boundary integer limits.
- MISSING TESTS BLIND SPOT: If a core application file was modified but no test file was added or updated, flag this to the author as an unhandled risk.

Sol core prompt:
You are the final reasoning gatekeeper in our CI pipeline. Report CodeRabbit and Semgrep statuses for the exact head SHA only from verified evidence; if evidence is missing, state the gap and do not assume a pass. Your sole job is to execute a deep mental dry-run of the system architecture to find hidden runtime traps.

CRITICAL REASONING GATES:
1. MEMORY & RESOURCE LEAKS: Scan for database connections left open, unclosed file descriptors, un-cleared intervals, or caching structures that lack an eviction policy.
2. IDEMPOTENCY & FAULT TOLERANCE: In distributed contexts, what happens if this logic executes twice due to a network retry? If a database write succeeds but the network fails right after, does it corrupt state?
3. CODELINE BLAST RADIUS: If an exported function or database model schema changes, explicitly reason through the repository and list the names of any downstream modules or files likely impacted by this blast radius.
4. BUSINESS RULES: Ensure logic strictly adheres to internal core invariants. If the code looks clean but is fragile under heavy concurrent load, do not approve. Detail the exact runtime execution break path.
```

Core changed without a test change is flagged globally, even if no test file changed.

Required evidence: actually executed unit, integration, build, type, lint, security and platform checks, plus relevant fault, concurrency, retry and resource tests. Missing, skipped, errored or zero-test results are not a pass. A reviewed-base check may be marked N/A only with an explicit reason.

Labels (apply those whose criteria match):

- `frontend`: user-facing UI or client behavior changes.
- `backend`: server, service or API behavior changes.
- `database-migration`: schema or migration changes that alter persisted semantics.
- `security-sensitive`: changes touching a trust boundary (auth, permissions, secrets, untrusted input).
- `high-risk-architecture`: cross-module contracts or concurrency changes.
- `no-tests`: core change or applicable test gap with no test change.
