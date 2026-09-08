You are building `assay` end to end, milestones 1 through 4, from `SPEC.md`.

This is an independent implementation. Another agent is building the same spec
in parallel. Do not read, import, copy, or inspect their work: not their branch,
not their files, not their commits, even if you can see them. The value of this
run is that two implementations were derived from the same document without
contact. Contaminate it and it is worth nothing.

Work on your own branch in your own worktree. Write only under paths you create.

Stack: Python 3.11+, Inspect AI (`inspect-ai`, developed against 0.3.263), `uv`.
Add `hypothesis` for property tests. Nothing else without recording why.

## Method: contract first, wiring last

`SPEC.md` is binding. Where this file and the spec disagree, the spec wins.

Do not follow a build order handed down to you. Derive it from the spec, and
start where the spec is most precise, which is scoring. In order of certainty:

1. The scoring rule and its four failure modes.
2. The dataset contract and the `requires` semantics.
3. Prompt assembly and the byte-identity guarantee across arms.
4. Skill loading and revision capture.
5. Inspect wiring.
6. Artifacts.
7. Reporting and the noise floor.

Build the scorer as a pure function of (completion, sample metadata, target)
with no Inspect types in its signature, then adapt it to Inspect at the edge.
It should be testable, and fully tested, before anything imports `inspect_ai`.

## Verification: properties, not just examples

Example tests are the floor. Above them, use `hypothesis` to state the
properties the spec actually implies, and let it hunt for the cases you would
not have written by hand. At minimum:

- A completion with no parseable verdict line is never CORRECT, for any sample,
  any metadata, any target.
- A control sample is CORRECT if and only if the verdict is CLEAN. No `requires`
  content can change that.
- On a defect sample, adding text to the named defect can only ever turn
  INCORRECT into CORRECT, never the reverse. Matching is monotone in what the
  model said.
- Adding an alternative to a `requires` group can only ever turn INCORRECT into
  CORRECT. Matching is monotone in the alternatives too.
- Adding a group to `requires` can only ever turn CORRECT into INCORRECT.
- Exactly one failure mode is recorded per incorrect score. They never stack and
  they are never empty.
- Case and surrounding whitespace never change a verdict.

Any property that fails is either a bug in your scorer or an ambiguity in the
spec. Both outcomes are findings. Write down which one it was.

## Fast mode: decide and record, do not block

You are running for throughput. Do not stop to ask on anything the spec leaves
genuinely open, and do not stall on a judgement call you can reverse later.
Choose the reading that keeps the measurement conservative, meaning the one
less likely to hand a point to the skill arm, then record it.

Keep `docs/ASSUMPTIONS.md` as you go. One entry per decision:

```
### <the question>
Spec says:      <the passage, quoted, or "silent">
I chose:        <the reading>
Because:        <one line>
Reversible:     yes | no, and what it would invalidate
```

Three things still stop you cold, because guessing on them makes every number
afterwards worthless while looking exactly as credible as a real one:

- A fixed parameter in the resolved parameters table appears wrong.
- The scoring rule cannot express a case you have actually hit.
- You are about to do something that changes what a published number means.

## Milestones

Gates are in the spec and they hold. Additionally, for this run:

- **M1** ends when the property suite passes and a mockllm run scores 0.0 with
  every sample recorded unparseable. Any other mockllm result is a scorer bug.
- **M2** ends with the human review pass on wrong-defect cases. You may not
  perform that pass yourself. Prepare the review packet, each case with the
  artifact, the planted defect, what the model said, and the group that missed,
  then stop and hand it over.
- **M3** ends with `docs/RESULTS.md` and its matching `docs/CRITIQUE.md` entry,
  the noise floor written down before the deltas were read, and skill over
  placebo as the headline.
- **M4** ends with the written finding on whether surface-form matching survives
  design critiques, per the spec, and no quiet loosening of the matcher if it
  does not.

## The deliverable nobody else produces

Alongside the working code, write `docs/DIVERGENCE.md`: every place `SPEC.md`
admitted more than one reasonable reading, what the readings were, and which you
took. This is the actual point of running two implementations. When the builds
are compared, a behavioral difference traced to an entry in this file is a spec
defect, and a difference not in this file is an implementation bug in one of us.

Be specific and unflattering. "The spec is clear" is not an entry. A spec with
no ambiguities has not been read hard enough.
