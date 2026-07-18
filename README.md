# Stonefold specifications

The canonical home of the Stonefold specs: the intent format an agent emits, the policy
language the gateway enforces, their JSON Schemas and worked examples, and the
specification of the conformance test kit that certifies any implementation against them.

One sentence for the whole stack: **the agent speaks SIF; Stonefold enforces; the rules
are carved in Stele — and nothing else can act.**

## What's here

| Artifact | Where | Status |
|---|---|---|
| **SIF** — the Structured Intent Format (the five action kinds + the shape the agent emits) | `docs/00-RFC-sif-intent-format.md` | v1.0 |
| **Stele** — the Stonefold policy language (*what's allowed*) | `docs/01-RFC-agent-control-policy.md` | v0.6 (changelog at top) |
| Change sets (deltas between Stele revisions; a change set wins on conflict with older wording) | `docs/RFC-changeset-*.md` | v0.1→v0.6 all accepted |
| Registry domain model (declaring resources, actions, states, scope predicates) | `docs/06-registry-domain-model.md` | |
| Artifacts & schemas (how the three schemas relate) | `docs/07-artifacts-and-schemas.md` | |
| Glossary (every concept in plain language) | `docs/08-glossary.md` | |
| Interception mapping (interpreting ordinary MCP/tool calls via the declared mapping) | `docs/17-interception-mapping.md` | |
| JSON Schemas for intents, policies, and registries | `schema/sif.schema.json`, `schema/stele.schema.json`, `schema/registry.schema.json` | |
| Worked policies + registries (fixtures; every one validates against its schema) | `examples/`, `registry/` | |
| **Conformance test kit (TCK)** — how ANY gateway, in any language, certifies against the RFCs | `docs/12-conformance-tck.md` (the runnable kit ships in the [reference repo](https://github.com/stonefold-ai/stonefold), `src/stonefold_tck/`) | |

This repo holds documents, schemas, and fixtures only — no code. The runnable TCK lives
in the reference repo, but its core imports nothing from the reference; it drives a
gateway either in-process (Python) or over the HTTP wire binding, so a Java/Go/Rust
implementation certifies the same way the reference does.

## Implementations

- **Reference implementation** (Python, plus the runnable demo and benchmark harness):
  [stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold) — certified by the
  TCK in-process and over the wire.

## Posture

Stonefold is an opinionated named product, not a committee standard: the action kinds,
gate types, and condition grammar are deliberately frozen (extensions live in resources,
actions, named sets, scope predicates, and hooks). The TCK is the standing answer to
"can others implement this without depending on the author." If a genuine working group
or a second independent implementation materializes, donating the spec under neutral
governance is on the table — from strength, not ahead of adoption.

## License

Apache-2.0 (see `LICENSE`). Anyone may implement this specification; no further
permission is needed. Provided as is, without warranty of any kind.
