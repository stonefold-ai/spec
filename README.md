# Stonefold documentation

The documentation of the Stonefold gateway: the wire format an agent speaks, the policy
language the gateway enforces, their JSON Schemas and fixtures, and the conformance test
kit that pins the gateway's guarantees.

One sentence for the whole stack: **the agent emits declared intents; Stonefold
enforces; the rules are carved in Stele — and nothing else can act.**

## How much of this is tested

This describes one implemented system: the reference gateway (Python, 552 automated
tests against real infrastructure), certified by the TCK in-process and over the wire.
Three estates — accounts payable, platform operations, and a results worklist — were
built as working systems, driven by real language models, and allowed to fail before the
controls were written; they are what earned v0.3.

Every gate in the catalog is implemented in the reference and covered by the kit. The
right-hand column below is the harder standard: whether a built estate, driven by a real
model, actually needed it.

| Gate | In the reference | Exercised by a tested estate |
|---|---|---|
| `rate` | yes | payments (per-payee rate) |
| `quota` | yes | none yet |
| `valueLimit` | yes | payments |
| `spendLimit` | yes | payments (daily total) |
| `allowlist` / `denylist` | yes | payments (sanctions), platform ops (static keys) |
| `precondition` | yes | all three estates |
| `contentCheck` | yes | none yet |
| `requireApproval` | yes | all three estates |
| `dualAuthorization` | yes | payments |
| `window` | yes | none yet |
| `quantityCap` | yes | none yet |
| `disclosure` | yes | none yet |
| `emissionControl` | yes | none yet |
| `requireExplanation` | yes | none yet |
| `requireMatch` | yes | payments (order line reserved and consumed) |

`examples/` holds fixtures from the tested estates only (plus one deliberately
invalid policy the linter must refuse). Fictional domain examples were removed on
purpose: an example whose estate was never built demonstrates syntax, and syntax is
what the schemas are for.

## What's here

| Artifact | Where | Status |
|---|---|---|
| **SIF** — the wire format the agent speaks (the five action kinds + the intent shape) | `docs/00-sif-wire-format.md` | v0.3 |
| **Stele** — the Stonefold policy language (*what's allowed*) | `docs/01-stele-policy-language.md` | v0.3 |
| Registry domain model (declaring resources, actions, states, scope predicates) | `docs/06-registry-domain-model.md` | |
| Artifacts & schemas (how the three schemas relate) | `docs/07-artifacts-and-schemas.md` | |
| Glossary (every concept in plain language) | `docs/08-glossary.md` | |
| Interception mapping (interpreting ordinary MCP/tool calls via the declared mapping) | `docs/17-interception-mapping.md` | |
| JSON Schemas for intents, policies, and registries | `schema/sif.schema.json`, `schema/stele.schema.json`, `schema/registry.schema.json` | |
| Worked policies + registries (fixtures; every one validates against its schema) | `examples/`, `registry/` | |
| **Conformance test kit (TCK)** — the black-box suite that certifies a gateway, in any language, against this specification | `docs/12-conformance-tck.md` (the runnable kit ships in the [reference repo](https://github.com/stonefold-ai/stonefold), `src/stonefold_tck/`) | |

This repo holds documents, schemas, and fixtures only — no code. The runnable TCK lives
in the reference repo, but its core imports nothing from the reference; it drives a
gateway either in-process (Python) or over the HTTP wire binding, so a Java/Go/Rust
implementation certifies the same way the reference does.

## Implementations

- **Reference implementation** (Python, plus the runnable demo and benchmark harness):
  [stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold) — certified by the
  TCK in-process and over the wire.

## Posture

This is the documentation of an opinionated product, not a committee standard, and the
version number measures its maturity honestly: one reference implementation, three
tested estates, no production deployments. The shape is deliberately frozen — five
action kinds, fifteen gates, a fixed condition grammar — and extension happens in
registries, not in the language. The TCK is the standing answer to "can others implement
this without depending on the author": the reference is certified by the same kit it
offers. If a second independent implementation materializes, neutral governance is on
the table — from strength, not ahead of adoption.

## License

Apache-2.0 (see `LICENSE`). Anyone may implement this specification; no further
permission is needed. Provided as is, without warranty of any kind.
