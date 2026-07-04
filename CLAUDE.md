# CLAUDE.md — Stonefold specifications

Conventions for any AI agent working in this repository: the **canonical home of the
Stonefold specs** — SIF, Stele, their JSON Schemas, worked examples/registries, and the
specification of the conformance TCK.

## What this repo is (and is not)

- **Documents, schemas, and fixtures only — no code.** Runnable artifacts (the reference
  gateway, the TCK kit, the registry generator, demos, benchmarks) live in
  [stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold). Never add code here.
- **This repo wins on divergence.** The stonefold repo carries working copies of these
  documents; edit here first (or sync here immediately), then copy to the stonefold repo's
  `docs/` so the trees stay byte-identical.

## Vocabulary (use it consistently)

**Stonefold** is the product; the enforcement component is the **gateway**; the policy
language is **Stele** (files `*.stele.yaml`, `apiVersion: stele/v0.1`); the intent format
the agent emits is **SIF**. One sentence: *the agent speaks SIF; Stonefold enforces; the
rules are carved in Stele — and nothing else can act.* Old names (ACP, Interlock,
agent-control-protocol) are fully retired; the rename record is `docs/renaming.md` in the
stonefold repo.

## Editing rules

1. **The shape is frozen.** Never add action kinds, gate types, attribute names, or
   condition operators (Stele RFC §13). Extensions go in resources, actions, named sets,
   scope predicates, and hooks — declared in registries, not in the language.
2. **Changes go through change sets.** Normative changes accumulate in the current draft
   `docs/RFC-changeset-*.md` (one `CS-nnn` item each: what/why/implementation impact) and
   are mirrored in the RFC's changelog table. A change set wins on conflict with older
   wording. The RFC header version bumps only when a set is closed/accepted.
3. **Schemas and fixtures move together.** Any schema change must keep every file in
   `examples/` and `registry/` valid against `schema/*.schema.json`; fixtures are the
   RFC's worked examples — keep them aligned with the RFC text they illustrate.
4. **RFC 2119 discipline.** MUST/SHOULD/MAY only in normative documents; supporting docs
   are context, not requirements. Document every feature as what/why/how for an
   RFC-reader implementer; don't duplicate normative text across documents — link to it.
5. **Referencing the product is fine; depending on it is not.** Documents may point at the
   stonefold repo for runnable artifacts (non-normative pointers). Normative content must
   stand alone so any implementation can be written from this repo.
