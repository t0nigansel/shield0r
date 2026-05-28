# shield0r — Product Vision

## One-liner

A runtime LLM guardrail proxy whose policies are derived from, and proven by,
real red-team findings — closing the loop between `hamm0r` (attack) and
`shield0r` (defense).

## Problem

LLM-backed systems are attackable through prompts: injection, jailbreaks,
system-prompt extraction, PII/secret egress, and tool-call abuse. Existing
guardrail tools detect a lot of this, but they share two gaps:

1. **No assurance loop.** Detection is asserted, not proven. There is no
   built-in way to demonstrate that a specific, observed attack is actually
   blocked — and stays blocked (no regression guarantee).
2. **Python-bound deployment.** The mature tools (LLM Guard, NeMo Guardrails,
   LiteLLM guardrails, LlamaFirewall, OpenGuardrails) are Python. That imposes a
   runtime, a dependency surface, and unpredictable tail latency on the hot path
   of every request.

## What shield0r is

A reverse proxy that speaks the OpenAI/Azure API shape. The target application
changes one thing — its base URL. No SDK, no code changes. shield0r inspects
both directions and enforces policy:

```
client -> [shield0r] -> LLM endpoint
              |              |
        inspect request  inspect response
              |              |
        allow / block / redact / flag (all logged)
```

- **Inbound (user -> LLM):** prompt injection, jailbreak, scope violation,
  user-side PII redaction.
- **Outbound (LLM -> user):** system-prompt leakage, PII/secret egress,
  tool-call / URL scope enforcement, unsafe content.

## Sibling tool: hamm0r

shield0r is the defensive half of a pair. `hamm0r`
(https://github.com/t0nigansel/hamm0r — canonical; the SpiritTesting repo is a
fork) is the offensive half:

- Rust + Tauri 2, single binary, local-first, no Python runtime.
- Fires OWASP-tagged attack prompts (OWASP Top 10 for LLM Applications 2025)
  against LLM endpoints as Cartesian-product matrix scenarios.
- File-based engagements under `~/hamm0r/engagements/<slug>/`: an
  `engagement.yaml`, append-only `runs/run-NNN.jsonl` run logs, optional
  `run-NNN.verdicts.jsonl` analyzer output, and raw response bodies.
- Optional local LLM judge (analyz0r) produces per-response verdicts.

Both tools being Rust means a shared language, shared types, and a clean
handoff artifact.

## What makes it different

### 1. The hamm0r <-> shield0r assurance loop (the core thesis)

```
hamm0r fires OWASP-tagged attacks at the LLM   ->   finds a bypass
        |
        v
   analyzer writes run-NNN.verdicts.jsonl
   (per-response verdict + OWASP ref + category)
        |
        v
shield0r imports the verdict stream and compiles it into a rule / policy
        |
        v
hamm0r re-runs   ->   attack now blocked   ->   regression locked in
```

No competitor owns both halves. This is a closed assurance cycle —
*measure exposure -> enforce -> prove enforcement -> prevent regression* — not
another standalone guardrail library. The **verdict-stream -> policy contract**
is the actual innovation and gets specified before any proxy code.

The contract is already half-built: hamm0r findings carry an OWASP ref and
category, so shield0r policies can key off the same taxonomy directly. The
handoff artifact is `run-NNN.verdicts.jsonl` — append-only, line-delimited,
trivially parsed in Rust with serde.

### 2. Honest Rust positioning

Rust is **not** claimed to make detection faster — detection latency is
dominated by classifier/LLM-judge inference, which shield0r delegates to the
best available models regardless of language. Rust is chosen for the
**enforcement and deployment layer**:

- Single static binary, no runtime, tiny container — deployable as a sidecar or
  at the edge where a Python stack is awkward.
- Predictable p99 / concurrency on the request hot path (`axum` / `tower`, no
  GIL contention).
- A memory-safe systems language is a credible story for regulated EU clients.
- Shares language and types with hamm0r — one stack across the attack/defense
  pair.

shield0r is **hybrid by design:** deterministic enforcement in Rust, smart
detection via existing models (e.g. Meta PromptGuard 2, gpt-oss-safeguard,
local/remote classifiers). We do not reimplement classifiers in Rust.

## Detection model: categories, not signatures

Coverage is organized around OWASP LLM Top 10 *intents*, not an enumeration of
attack strings — the same taxonomy hamm0r already tags prompts with. Each
category is a detector module routed through tiered enforcement:

| Layer | Good at | Role |
|-------|---------|------|
| Deterministic (regex/denylist) | literal strings, encodings, secret formats | stays small on purpose; obvious block/allow |
| Classifier (local/remote model) | semantic intent, paraphrase | absorbs novelty |
| LLM judge (optional) | reasoning, grey-zone context | escalation only, never the default gate |

Decision flow per request: cheap checks decide most traffic outright (block or
allow); the judge is consulted **only** for the ambiguous middle. Grey-zone
behavior on judge timeout is configurable: `fail-closed` (high security) or
`fail-open` (availability).

Detectors are **versioned data, not code** — a `shield0r-rules` pack feed,
updatable without recompiling, contributable by clients/community, and tested by
hamm0r.

## Competitive landscape (clear-eyed)

- **LLM Guard / LiteLLM** — own the "drop-in proxy + modular scanners" slot.
- **LlamaFirewall (Meta) / OpenGuardrails** — own the serious-research slot;
  ship strong open classifiers and target agent-layer risk.
- **PromptGuard 2 / gpt-oss-safeguard** — free off-the-shelf classifiers.

shield0r does **not** compete on breadth of pre-built rails, ecosystem, or
classifier quality. It competes on: (a) the assurance loop nobody else has, and
(b) Rust deployment/tail-latency for teams that want it. A me-too proxy would
lose; the loop + ownership of both halves is the defensible position.

## Non-goals

- Not a from-scratch injection classifier (use existing models).
- Not a dialog-flow framework (NeMo's space).
- Not breadth-of-rails parity with LLM Guard.
- Not "Rust because Python is for amateurs" — that framing is rejected.

## Open design questions (to resolve before/while building)

1. **Verdict-stream -> policy schema** — the keystone contract. What fields in
   `verdicts.jsonl` let shield0r auto-derive a rule that survives paraphrase?
   (OWASP ref + category are present; what else is needed — example payloads,
   target detector layer, suggested threshold?)
2. **Fix taxonomy** — which findings map to a rule-pack update vs. a
   threshold/model gap with no clean rule fix; what shield0r does in that gap.
3. **Proxy placement default** — transparent inline (preferred) vs. explicit
   sidecar/library call.

## Build order

1. Verdict-stream -> policy **contract spec** (keystone), designed against
   hamm0r's real `verdicts.jsonl` format.
2. Proxy skeleton: `axum` transparent reverse proxy forwarding OpenAI/Azure
   calls through an empty policy chain.
3. Deterministic detector layer + rule-pack format.
4. Classifier integration (wrap existing model).
5. hamm0r verdict import -> policy compiler -> replay regression.
6. Optional LLM-judge escalation tier.

## Success criteria

shield0r is working when: a hamm0r engagement that finds a bypass can be
imported from its `verdicts.jsonl`, compiled into a policy, and re-run to show
the same attack blocked — automatically, with the regression captured as a
repeatable test.