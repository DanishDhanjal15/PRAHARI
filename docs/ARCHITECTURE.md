# PRAHARI — architecture

> Design document. The prototype is built during the 36-hour finale; this describes the target design.

## Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  OPS          FastAPI control plane · HTMX dashboard · SQLite   │
│               structured JSON audit log · Docker Compose        │
├─────────────────────────────────────────────────────────────────┤
│  ASSURANCE    pytest harness · differential replay · git         │
│               apply/rollback · ed25519 proof-bundle signer       │
├─────────────────────────────────────────────────────────────────┤
│  REASONING    asyncio orchestrator (explicit state machine)      │
│               Tier-1 local: Ollama + Qwen2.5-Coder 3B/7B         │
│               Tier-2 cloud: Claude API (escalation only)         │
│               FAISS RAG over CWE/CVE + codebase                  │
├─────────────────────────────────────────────────────────────────┤
│  SENSING      Semgrep · CodeQL · tree-sitter · OpenAPI parser    │
│               Atheris · Hypothesis · schemathesis · coverage.py  │
│               PySan taint runtime                                │
├─────────────────────────────────────────────────────────────────┤
│  TARGET       Flask · FastAPI · Django · Node, containerised     │
└─────────────────────────────────────────────────────────────────┘
```

## Orchestrator

An explicit `asyncio` state machine, not an open-ended agent loop. Every transition is a named state
with defined inputs, outputs and failure edges, and every transition is written to the audit log.

This is a deliberate design choice. An autonomous system that modifies source code inside a defence
network has to be *auditable after the fact*, which means its control flow must be inspectable rather
than emergent. The LLM is called at specific, bounded points inside the state machine; it does not
drive the state machine.

States:

```
DISCOVER → RANK → HARNESS → EXECUTE → TRIAGE → PATCH → PROVE → {ACCEPT | RETRY | ESCALATE}
                                                   ▲            │
                                                   └── RETRY ───┘
```

`RETRY` re-enters `PATCH` with the failing gate's evidence appended to the context, bounded at *K*
iterations (default 3). Exhausting *K* moves to `ESCALATE`: the finding, the PoV and every rejected
candidate are handed to a human with the reason each one failed.

## Two-tier reasoning and the escalation policy

The cloud is an optional escalation, never a dependency. The system must complete a full loop with the
network cable unplugged.

| Task | Tier | Why |
| --- | --- | --- |
| Triage, dedup, CWE mapping, severity scoring | Local 3B | Classification over short structured input |
| Harness synthesis from a route spec | Local 3B/7B | Templated generation, spec-constrained |
| Patch synthesis, candidates 1–N | Local 7B | Minimal diffs constrained by a known taint path |
| Patch synthesis after local candidates fail all gates | **Cloud** | Genuinely hard reasoning, rare by design |
| Root-cause explanation for the escalation report | **Cloud** | Human-facing prose, quality matters |

**Target: ≥90% of decisions resolved on-device.** The orchestrator meters tokens and wall-clock per
stage and writes both to the audit log, so the local/cloud split is a measured number rather than a
claim. When the system is run air-gapped, tier 2 is simply disabled and exhausted candidates go
straight to `ESCALATE`.

## Context economy

The reason a 3B model is viable at all is that it is never asked to read a repository.

Stage 02 ranks source-to-sink paths statically, so by the time the model is invoked the input is a
handful of code slices along one candidate path, plus the taint path from the PoV. This keeps prompts
small, keeps latency low, and is what makes the whole system fit inside a 6 GB memory target.

## Patch synthesis constraints

A candidate patch is generated against a hard constraint set, not a free-form instruction:

- It may only modify files on the recorded taint path.
- It must be a minimal diff; whole-file rewrites are rejected before they reach the gates.
- It must not modify tests (that would trivially satisfy G2).
- It must not modify the PySan configuration (that would trivially satisfy G1 and G4).

The last two matter more than they look. An autonomous patcher scored on "do the tests pass" will, given
enough freedom, learn to edit the tests. Making those files immutable to the patcher is what keeps the
gates meaningful.

## The four gates

| Gate | Check | Failure means |
| --- | --- | --- |
| **G1** | Replay the PoV against the patched target; the taint path must no longer reach the sink | The patch does not fix the bug |
| **G2** | Full existing test suite, 100% pass | The patch broke declared behaviour |
| **G3** | Differential fuzz: 10,000 benign requests against pre- and post-patch instances; responses must be identical | The patch broke undeclared behaviour |
| **G4** | Re-run PySan across the full corpus; no taint path may exist that was absent before | The patch introduced a new vulnerability |

G3 is the gate that catches the failure mode nobody tests for: a patch that closes the injection by
over-sanitising, and quietly breaks every legitimate request carrying an apostrophe. Regression suites on
real codebases do not cover this. Differential replay does.

All four must pass. There is no partial credit and no confidence threshold.

## Proof bundle

On `ACCEPT`, the assurance layer emits a signed bundle: the finding, the PoV, the patch diff, the four
gate results with their evidence, and the environment fingerprint — signed with ed25519.

The verifier is deliberately a separate, small program with no dependency on the reasoning layer. A
reviewer who does not trust PRAHARI can re-run the bundle against the target and check every claim
independently. Schema: [`proof-bundle.schema.json`](proof-bundle.schema.json).

## Deployment

One `docker compose up`. The target runs in its own container; PRAHARI runs in another; the local model
is served by Ollama on the host or in a third container. No inbound network access is required and no
source code leaves the network.

Pointing it at a target requires exactly two arguments:

```
prahari run --repo ./target --url http://127.0.0.1:8000
```

There are no per-application rules to write and no retraining step, which is what makes it deployable
against infrastructure the team has never seen.
