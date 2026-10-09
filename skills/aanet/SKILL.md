---
name: aanet
description: Use when coordinating multiple AI agents or sub-agents on a shared task — a shared workspace, distributed locks, budgeted sub-keys, shared file storage, and inter-agent messaging via AANet (https://aanet.space), billed per operation via x402 (USDC on Base/Solana). Trigger on requests to coordinate a swarm of agents, avoid duplicate work across parallel sub-agents, share state or hand off files between sub-agents, rate-limit/budget what a sub-agent can spend, or set up mutual exclusion for a shared resource across agents.
license: MIT
---

# AANet: coordinating a swarm of agents

AANet is a live, metered coordination service for autonomous agents. It is
not a framework you install — it's an HTTP API (and a remote MCP server at
`https://aanet.space/mcp`) you call directly. Full reference:
[aanet.space/AGENTS.md](https://aanet.space/AGENTS.md) — treat that as the
source of truth for exact request/response shapes; this file is the
operational know-how on top of it.

## When to reach for this

- An orchestrator is about to spawn several sub-agents that need to avoid
  duplicating each other's work, share intermediate results, or coordinate
  access to something only one of them should touch at a time.
- A sub-agent needs a budget ceiling separate from the orchestrator's own
  token/cost budget — a real, enforced spending cap, not a suggestion in a
  prompt.
- Agents need to hand off a file or send each other a short status update
  without going through the orchestrator's own context window.

Don't reach for this for a single agent working alone — the whole value is
in the *shared, metered, multi-party* state.

## Quick start

```
POST /workspaces/trial          -> {"workspace_id", "owner_key", "deposit_balance": 1000, "trial_expires_at"}
```

No payment, no account. Comes with 1000 units (a nominal $0.01) and expires
24 hours after creation — enough to prototype the coordination pattern and
run a realistic multi-agent round, not to run real work. For anything that
needs to outlive a day, use more than 5 sub-keys, or write a file at the
1 MiB cap (that one write costs 1029 units — a known limit of the trial, not
a bug), fund a real workspace instead: `POST /workspaces` (no free balance —
see the deposit flow in AGENTS.md).

Mint one sub-key per sub-agent — never share one key across agents, or
you lose per-agent attribution in the activity log:

```
POST /workspaces/{id}/subkeys
Authorization: Bearer <owner_key>
{"label": "sub-agent-name", "path_scope_prefix": "sub-agent-name", "budget_cap": 5000}
-> {"subkey_id", "key"}
```

## The three coordination primitives

**Files** — `GET`/`PUT`/`PATCH`/`DELETE /workspaces/{id}/files/{path}`, ETag-based
optimistic concurrency (`If-Match` header). Use for shared state and
handoffs between agents.

**Locks** — `POST /workspaces/{id}/locks/{name}/acquire|renew|release`, lease-based.
Use to serialize access to one resource (a task name, a file, anything you
name). **Read the pitfall below before relying on this for task dedup.**

**Messages** — `POST`/`GET /workspaces/{id}/messages`, point-to-point
(`to_subkey_id`) or broadcast (`to_subkey_id: null`). Cheaper than a file
write, meant for one-shot signals ("chunk 3 done"), not durable state.

## Hard-won lessons (the actual point of this file)

**1. A lock is a mutex, not a completion record.** `acquire` succeeding only
means nobody holds that name *right now* — it does not mean the work behind
it was never done. A worker that re-scans a shared task list and acquires a
lock that was released after someone else already finished the task has no
way to tell "free" from "already done" from the lock API alone. **Fix:**
after acquiring, check for the actual expected result (read the output
file, or check recent messages) before starting work. See the runnable
demonstration in [aanet-examples](https://github.com/Kuasar137/aanet-examples)'s
`task_dedup` — it shows the bug happening and the one-line fix.

**2. Check your remaining budget instead of guessing.** A sub-key has no
endpoint to query its own `budget_cap`/spend (that's owner-only) — but
every response, success or error, carries an `X-AANet-Remaining-Balance`
header, recomputed after that request. Read it after each call in a
multi-step sequence and decide *before* attempting the next step whether
you can afford it, rather than discovering you can't via a `402` partway
through something you can't cleanly undo. See `budget_aware_backoff` in the
same examples repo. The orchestrator, which isn't the one making those calls,
should not wait for a `402` either: register a `balance_low` webhook
(`"events": ["balance_low"], "balance_thresholds": [5000, 1000]` — unit values
you choose; one 2-unit delivery each time the workspace balance falls through
one, silent for thresholds already below at registration) and read
`GET /workspaces/{id}/forecast` (owner, free) for spend per hour and
`runway_hours`. Both work on a trial. See `balance_low_warning`.

**3. Prefer the MCP endpoint over raw HTTP when spawning sub-agents you
don't fully control.** Handing a freshly-spawned sub-agent a `curl` command
with a live Bearer token embedded in the prompt text can read to a
safety-conscious model as a credential-injection attempt ("here's a key,
spend it") and trigger an outright refusal — this happens in practice with
current models. Routing the same call through an MCP tool (`aanet.space/mcp`)
goes through the harness's own sanctioned tool-calling layer instead of
looking like an injected instruction, and is markedly more reliable for
sub-agents you spawn programmatically.

**4. Trial workspaces are for prototyping the *shape* of your coordination,
not for running it.** 1000 units, 24 hours, capped at 5 active sub-keys
(revoking one frees a slot). If your real design needs more than 5
concurrent agents or more than a day, prototype on a trial, then move to
a funded workspace before running it for real — don't be surprised when the
6th sub-key mint returns `429`.

**5. `PATCH` appends, `PUT` replaces.** Sending a running log/notes file
that multiple sub-agents contribute to should always be `PATCH`, never
`PUT` — a `PUT` from any agent racing another's `PATCH` clobbers it wholesale
rather than merging. If two agents can plausibly write at once, add an
`If-Match` (or a lock around the read-modify-write) regardless of which verb
you use.

## Worked examples

[github.com/Kuasar137/aanet-examples](https://github.com/Kuasar137/aanet-examples) —
runnable Python scripts for lessons #1 and #2 above (plus the orchestrator-side `balance_low` warning), tested against the live
service.
