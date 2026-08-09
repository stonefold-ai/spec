# Contributing to the Stonefold specifications

Thanks for your interest. This repo is the **canonical home of the Stonefold specs** —
SIF, Stele, their JSON Schemas, worked examples and registries, and the specification of
the conformance TCK. The reference gateway, the runnable TCK, demos, and tooling live in
[stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold); implementation
issues belong there.

Please read this document before opening a PR. Stonefold is a security project: its value
rests on a precisely worded specification, so contributions work differently here than in
a typical "PRs welcome" repository.

**The one-line summary: we agree on intent before we review changes.**

## Why these rules exist

Producing a large, plausible-looking pull request now takes hours. Reviewing one still
takes real human attention, and an unreviewed change to a security specification is
attack surface for every implementation downstream. Review capacity is the scarcest
resource in this project; these rules exist to spend it where it matters.

## Posture

Stonefold is an **opinionated named product, not a committee standard** (see the README's
Posture section). The action kinds, gate types, and condition grammar are deliberately
frozen; proposals to extend the *language* will be declined. That is an architectural
rule, not gatekeeping: extension happens in the declared vocabulary — resources, actions,
named sets, scope predicates, and hooks — which is exactly what registries are for. Most
"extend the language" ideas are registry ideas that didn't know where the door was;
expect to be redirected there, with a sketch.

## The golden rule: proposal before pull request

**Unsolicited pull requests making sizeable normative changes will be closed without
review.** The flow for anything non-trivial:

1. **Open an issue describing your intent** — what you want to change, why, and how it
   affects (or provably does not affect) the stated guarantees. Cite the specification section
   (`§n`) and, if relevant, the schema or fixture that disagrees.
2. **Wait for an explicit go-ahead** from the maintainer.
3. **Then write the change.** Trivial fixes (typos, broken links, wrong cross-references)
   skip the proposal step.

## Two lanes for spec contributions

This repo is not closed; it is gated by evidence.

**Lane A — problems. Open to everyone, no proposal needed, and the most valuable
contribution there is.** Contradictions between the specification text, the JSON Schemas, and the
fixtures; ambiguities where two independent implementations could reasonably disagree; fixtures — worked policies or
registries — that expose an under-specified corner; attacks that break a stated guarantee
(those go through [`SECURITY.md`](SECURITY.md), privately). Finding a real hole requires
no alignment with the maintainer's design intent — it only requires the hole to be real.
The spec is pre-1.0 precisely because demonstrated holes are expected to reshape it
before it stabilises.

**Lane B — design changes. Start from a problem, not a solution.** A proposal to change
normative text must name the demonstrated hole it fixes, or the real thing the current
design cannot express. If the maintainer agrees the problem is real, the fix is worked
jointly, and you can author it.
Proposals that arrive as a finished design with no agreed problem behind it will be
declined regardless of quality.

**Declines cite the criterion.** Every declined proposal states which test failed
("problem not demonstrated", or "expressible today via X") and stays public in the issue,
so the project's direction is legible before you invest work.

## How changes land

1. Open an issue as described above.
2. Normative changes are edited directly into the specification. The version bumps
   when a coherent set of changes lands; the git history is the change record.
3. PRs must keep every file in `examples/` and `registry/` valid against the schemas in
   `schema/`, and must not use RFC 2119 keywords outside normative documents.

## What does not belong here

- **Code.** The reference implementation, the runnable TCK, demos, and tooling live in
  [stonefold-ai/stonefold](https://github.com/stonefold-ai/stonefold). Attack-scenario
  test cases — the most wanted code contribution — go there too, in the format of its
  `tests/acceptance-scenarios.md`.
- **The control plane.** Policy distribution, multi-gateway coordination, and environment
  management do not exist yet; their design is reserved to the maintainer for now. Issues
  and design feedback are welcome; pull requests will be declined regardless of quality.

## LLM-assisted contributions

Using LLMs to draft text, fixtures, or reports is fine here; the maintainer does too.
What is not fine:

- Submitting generated text you have not personally read, understood, and checked against
  the schemas and fixtures. **You are the author of record.**
- Generated issue or security reports that no human has verified. Unverified AI-generated
  reports waste review capacity and will get you banned faster than anything else.

## Certifying an implementation

You don't need to contribute here to build a gateway: `docs/12-conformance-tck.md`
describes how any implementation, in any language, certifies against the specification using the
TCK (shipped in the stonefold repo). A conformance claim names the profiles certified and
the kit version. The kit has so far been run against exactly one gateway — the reference;
being the first independent implementation to certify would be a milestone for the
project.

## License

Apache-2.0. By contributing you agree that:

- your contribution is licensed under Apache-2.0 (§5);
- you certify the [Developer Certificate of Origin](https://developercertificate.org/)
  and sign each commit with `Signed-off-by` (`git commit -s`);
- you grant the maintainer the right to distribute future versions of the specifications,
  including your contribution, under other license terms.

That last point keeps the project's licensing options open without needing to track down
every past contributor. It does not change the terms of anything already released: every
version published under Apache-2.0 stays Apache-2.0 forever, and the right to implement
the specifications is unconditional (see the README).

## What to expect from the maintainer

This is currently a solo-maintained project. In return for following the rules above, you
can expect a response to proposals within a reasonable time, honest reasons for any
rejection, and credit for your contributions. What you should not expect is fast review
of large or unsolicited work — that trade is what this document is for.
