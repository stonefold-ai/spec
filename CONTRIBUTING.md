# Contributing to the Stonefold specifications

Thanks for your interest. A few things to know before opening an issue or PR — they will
save you a round-trip.

## Posture

Stonefold is an **opinionated named product, not a committee standard** (see the README's
Posture section). The action kinds, gate types, and condition grammar are deliberately
frozen; proposals to extend the *language* will be declined. Extension happens in the
declared vocabulary — resources, actions, named sets, scope predicates, and hooks — which
is exactly what registries are for.

## What contributions fit here

- **Spec defects:** contradictions between the RFC text, the JSON Schemas, and the
  fixtures; ambiguities an implementer cannot resolve; wrong or missing cross-references.
- **Clarifications:** places where two independent implementations could reasonably
  disagree — these are the highest-value reports (see the changelog's CLARIFIED items for
  the shape).
- **Fixtures:** worked policies/registries that expose an under-specified corner.
- **Code does not belong here.** The reference implementation, the runnable TCK, demos,
  and tooling live in [stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold);
  file implementation issues there.

## How changes land

1. Open an issue describing the defect/ambiguity, citing the RFC section (`§n`) and, if
   relevant, the schema/fixture that disagrees.
2. Normative changes are recorded as **change-set items** (`CS-nnn` in the current draft
   `docs/RFC-changeset-*.md`, mirrored in the RFC changelog table) — each states *what*,
   *why*, and *implementation impact*. A change set wins on conflict with older wording;
   the RFC version bumps only when the set closes.
3. PRs must keep every file in `examples/` and `registry/` valid against the schemas in
   `schema/`, and must not use RFC 2119 keywords outside normative documents.

## Certifying an implementation

You don't need to contribute here to build a gateway: `docs/12-conformance-tck.md`
describes how any implementation, in any language, certifies against these RFCs using the
TCK (shipped in the stonefold repo). A conformance claim names the profiles certified and
the kit version.

## License

Apache-2.0. By contributing you agree your contribution is licensed under the same terms
(Apache-2.0 §5).
