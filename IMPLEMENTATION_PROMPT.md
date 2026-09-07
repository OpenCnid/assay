You are building `assay` end to end, milestones 1 through 4.

`SPEC.md` in this repository is binding. Read it before you write anything, and
read it again at the start of each milestone. Where this file and the spec
disagree, the spec wins: stop and flag the conflict rather than resolving it
silently. Everything the spec fixes (the arms, the dataset contract, the scoring
rule, the resolved parameters table, the milestone gates) is settled and not
yours to renegotiate mid-build. If you believe a fixed parameter is wrong, say
so and wait.

Stack: Python 3.11+, Inspect AI (`inspect-ai`, developed against 0.3.263), `uv`
for dependency management. No further runtime dependencies without asking.

## How to work

Milestones are sequential and gated. The gate for each is in `SPEC.md`. Do not
begin a milestone until the previous gate is met, and do not report a gate met
on the strength of code that has not run.

Record progress in `docs/PROGRESS.md` as you go: what landed, what the gate
evidence was, and what you decided where the spec left room. A future session
picks up from that file, so write it for someone with none of your context.

Two rules that hold in every milestone:

- Never let the model grade the model. Not for scoring, not for dataset
  validation, not as a convenience during development.
- Never reproduce Dovetail's skill text in this repository. It is read at run
  time from a checkout, per the skill loading contract.

## Milestone 1: harness

Build in this order.

1. `src/assay/arms.py`. Load a named skill from a Dovetail checkout path supplied
   as a task parameter, following the resolution order and the loud-failure rules
   in the spec. Capture the checkout's git revision and dirty state and return
   them with the text so both reach the eval log. Assemble the three preambles,
   with the shared instruction and response protocol block byte-identical across
   arms. Assert that identity in code, not just in a test: it is the assumption
   the whole comparison rests on.

2. `artifacts/judge/`. One file per artifact, the six defect classes and two
   controls named in the spec. Write them yourself rather than lifting public
   code, and keep each short enough to review by eye. Exactly one planted defect
   per artifact; if you find yourself planting a second, split it in two.

3. `src/assay/dataset.py`. Load artifacts from disk into Inspect samples carrying
   the full metadata contract. Write each `requires` group before you read the
   skill text for that class, per the author-bias mitigation, and note in
   `docs/PROGRESS.md` that you did. Be generous with alternative surface forms:
   a false INCORRECT is worse than a permissive match, because it silently
   deflates whichever arm it lands on.

4. `src/assay/scoring.py`. The deterministic scorer and the metrics, exactly as
   specified. Distinguish unparseable, miss, false positive, and wrong-defect in
   score metadata, and keep them unblended in the reported metrics.

5. `src/assay/tasks.py`. Parameterized tasks taking the suite, the arm, and the
   Dovetail path. Reject any arm value outside the three named. Name each task by
   suite and arm so logs are distinguishable without opening them.

6. `tests/`. The scorer is the part that must not be wrong: cover a correct hit,
   a wrong-defect hit, a miss, a false positive, and an unparseable response,
   plus every skill-loading failure and the protocol-identity assertion.

Gate evidence:

```
uv run pytest
uv run inspect eval src/assay/tasks.py -T suite=judge -T arm=baseline --model mockllm/model
```

The mock model produces no parseable verdicts, so 0.0 with every sample recorded
unparseable is the correct outcome and confirms the scorer gives away nothing.
Any other result under mockllm is a bug in the scorer, not a curiosity.

Then run both arms against one real model as a pilot. The numbers are not a
result and do not go in `docs/RESULTS.md`.

## Milestone 2: judge dataset

Grow to 60 defect and 20 control artifacts, difficulty labeled, no class over
20% of the set. Extend the six pilot classes rather than inventing a wholly new
taxonomy, and keep the obvious-to-subtle spread deliberate: a set that drifts
subtle measures a different thing than the one specified.

Run the full pilot, then do the human review pass the gate requires. Every
wrong-defect case gets read by a person before anything is published. Where the
model named the planted defect in words no `requires` group anticipated, widen
that group once, then freeze it. Do not widen in response to a failure seen
mid-run; record it, finish the run, widen between runs.

Log every widening in `docs/PROGRESS.md` with the phrasing that prompted it.
That log is what tells a reader later whether the dataset was tuned toward a
result.

## Milestone 3: first result

Add the placebo arm: length-matched generic review advice carrying none of
judge-composition's structure. Match on token count, and record both counts.

Run five epochs per arm across at least two model families from different
vendors, at a pinned Dovetail revision with a clean checkout. Build
`src/assay/report.py` to compare arms, report variance across epochs, and
compute the noise floor for the dataset size. Compute the floor before you look
at the deltas, and write it down first.

Publish to `docs/RESULTS.md`: the date, the Dovetail revision, the models, the
four metrics per arm, and the skill-over-placebo delta as the headline.
Skill-over-baseline may appear beside it, labeled as an uncontrolled upper
bound. Write the matching `docs/CRITIQUE.md` entry in the same commit, covering
every threat in the spec that applies, including author bias.

If the delta is at or below the noise floor, that is the result. Report it
plainly. A null result honestly reported is the outcome this benchmark exists to
be capable of producing, and reaching for a reason to keep running until the
number improves is the exact failure the whole design is built to prevent.

## Milestone 4: self-play

Second suite, same harness, same scoring, new artifact class: short design notes
whose flaw is invisible as written and manifests only under a scenario the
design does not mention. 30 flawed, 10 sound. Sound controls must survive
stress, not merely look unremarkable.

Reuse `arms.py` unchanged by passing the skill name through. If self-play's
skill directory does not follow the same layout, extend the resolver rather than
special-casing it.

The open question you are expected to answer here, not assume: whether a latent
design flaw can be scored by the same surface-form matching that works for code
defects. Design critiques are more varied in phrasing than "off by one", so the
`requires` approach may not survive contact with this suite. If it does not, say
so, write up why in `docs/CRITIQUE.md`, and propose an alternative that still
involves no model grading. Do not quietly loosen the matcher to make the numbers
work.

Gate is the same as milestone 3, plus that written finding.

## When to stop and ask

- A fixed parameter in the spec looks wrong.
- The scoring rule cannot express a case you have hit.
- A milestone gate cannot be met with the design as specified.
- You are about to make a choice that would change what a published number
  means.

Stopping to ask on those is correct and expected. Guessing on them is the one
failure mode that makes the whole benchmark worthless, because every number it
produces afterwards looks exactly as credible as a real one.
