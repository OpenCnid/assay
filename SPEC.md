# assay

Measuring whether agent self-verification skills actually work.

## Status

Specification. No implementation exists yet. Two throwaway sketches confirmed
the harness shape is viable under Inspect AI 0.3.263; they are not the
implementation and are not in this repository.

This spec covers the whole program, milestones 1 through 4. `IMPLEMENTATION_PROMPT.md`
is the build order. Where the two disagree, this file wins.

## The claim under test

Dovetail (github.com/OpenCnid/dovetail) ships eight skills whose stated purpose
is to make Claude Code and Codex better at checking their own work. That claim
is currently unmeasured. No delta has ever been published for any of the eight.

The program measures two of them, in order.

**Judge-composition (milestones 1 to 3):**

> Loading the `judge-composition` skill increases an agent's ability to
> correctly identify a planted defect in an artifact, without a matching
> increase in false positives on clean artifacts.

**Self-play (milestone 4):**

> Loading the `self-play` skill increases an agent's ability to surface a latent
> flaw in a design, one that only manifests under a scenario not stated in the
> design, without a matching increase in false alarms on sound designs.

The remaining six skills are out of scope. A result about two skills is not a
result about Dovetail.

## Method

An ablation over one dataset, one model, one response protocol. The only thing
that varies between arms is the system prompt preamble.

| Arm | Preamble |
| --- | --- |
| `baseline` | None |
| `placebo` | Length-matched generic review advice, none of the skill's structure |
| `skill` | The skill text, read from a pinned Dovetail revision |

All three arms then carry a byte-identical instruction and response protocol.

The headline result is **skill over placebo**. Skill over baseline may be
reported alongside it, labeled as an upper bound that has not controlled for
prompt length, and never on its own.

Two numbers move together or the result is not what it looks like. A skill that
raises defect recall and false positives equally has not improved judgement, it
has made the model more suspicious. Report both or report neither.

## Resolved parameters

These were open questions. They are now fixed, and changing one invalidates any
result already published under the old value.

| Parameter | Value |
| --- | --- |
| Judge dataset size | 60 defect, 20 control |
| Self-play dataset size | 30 flawed designs, 10 sound |
| Defect to control ratio | 3:1 |
| Epochs per arm | 5 |
| Model families | At least two, from different vendors, named in run config |
| Dovetail revision | Pinned per published run, recorded in the log |
| Artifact storage | One file per artifact under `artifacts/` |
| Arms in the published claim | skill vs placebo |

Pilot runs during milestone 1 may use a smaller dataset and one epoch. A pilot
number never goes in `docs/RESULTS.md`.

## Skill loading contract

Skill text is never reproduced in this repository. It is read at run time from a
Dovetail checkout so the benchmark always measures what actually ships.

- Resolve `skills/<skill-name>/SKILL.md`, falling back to `skill.md` then
  `README.md`.
- Fail loudly on a missing directory, a missing skill file, or an empty one.
  A silently empty skill makes the arms identical and reports a delta of zero
  for the wrong reason. This is the most likely way for this benchmark to
  produce a confidently wrong result.
- Record the resolved Dovetail git revision, and whether the checkout was dirty,
  in the eval log metadata. A result that cannot name the skill revision it
  measured is not a result.
- A dirty checkout is permitted in development and disqualifying for a published
  run.

## Dataset contract

A sample is an artifact plus the ground truth about what is wrong with it.

```
id                 stable slug, e.g. "off-by-one-retry"
input              the artifact under review, verbatim
target             "DEFECT" or "CLEAN"
metadata.suite     "judge" or "selfplay"
metadata.kind      "defect" or "control"
metadata.planted   one sentence describing the planted defect, or "none"
metadata.requires  list of alternative-groups (see scoring)
metadata.difficulty  "obvious" | "moderate" | "subtle"
```

Every defect sample carries exactly one planted defect. An artifact with two
plausible serious defects is ambiguous and belongs in the dataset only once the
scoring rule can express that.

Controls are load bearing, not filler. A judge that answers DEFECT on every
sample scores 100% on a defect-only set, and the benchmark measures nothing.

Difficulty is recorded so recall can be broken down. A skill that only helps on
obvious defects is a different finding from one that helps on subtle ones, and
the blended number hides which you have.

### Judge suite defect classes

Pilot classes, spanning obvious to subtle. The full set of 60 extends these,
with no class exceeding 20% of the dataset.

1. Off-by-one in a bounded loop (an exclusive range consuming the budget).
2. Unbounded concurrency (fan-out with no limit).
3. Check-then-act race (TOCTOU on a lease file).
4. Degenerate scorer (an empty rubric returning a perfect score).
5. Swallowed failure (a bare except that reports success anyway).
6. Timezone naivety (naive `utcnow` against a possibly aware value).

Controls: correct chunking with a guarded size argument, and a design note
describing an atomic idempotent export.

Class 4 is deliberate. A rubric that rewards an empty answer is the same failure
shape the whole exercise is about.

### Self-play suite

Artifacts are short design notes rather than code. The flaw must not be visible
in the design as written; it manifests only under a scenario the design does not
mention. A retry policy that is correct until the downstream service becomes
non-idempotent, a cache invalidation that is correct until two writers land in
the same second, a migration that is correct until it is resumed after a partial
failure.

Sound controls must be genuinely sound under stress, not merely unremarkable.

## Response protocol

Identical across all arms, byte for byte. Deterministic scoring depends on it.

```
VERDICT: DEFECT | CLEAN
DEFECT: <one sentence naming the single most serious defect, or "none">
```

Report CLEAN only when there is no material defect. Style, naming, and missing
tests are explicitly not material defects; without that line, controls collect
false positives for reasons that have nothing to do with judgement.

## Scoring

Deterministic. No model grades the model, at any milestone. This is not a purity
preference: a model grader would be the largest uncontrolled variable in an
experiment whose entire output is a small delta.

- Unparseable verdict line: INCORRECT, recorded distinctly from a wrong answer.
  Protocol compliance is itself a measurement, and a longer preamble may change
  it.
- Control sample: CORRECT if and only if the verdict is CLEAN. A DEFECT verdict
  is recorded as a false positive.
- Defect sample with a CLEAN verdict: INCORRECT, recorded as a miss.
- Defect sample with a DEFECT verdict: CORRECT if and only if every group in
  `requires` is satisfied. Each group is a set of alternative surface forms for
  one idea, so "off by one" and "one fewer retry" both count, while naming some
  other real bug does not. Recorded as a wrong-defect hit when it fails.

### Metrics

Report separately, never blended into one accuracy:

- Defect recall, overall and split by difficulty
- False positive rate on controls
- Wrong-defect rate
- Protocol compliance

## Threats to validity

These belong in every published result, not only here.

1. **Keyword brittleness.** A correct answer phrased outside every `requires`
   group scores as wrong. A human reads every wrong-defect case before a delta
   is published, and alternates are widened in the dataset rather than the
   scorer being loosened. Widen between runs, never mid-run.
2. **Prompt length confound.** Addressed by the placebo arm. A result reported
   without it is an upper bound, not a measurement.
3. **Sample count.** A delta below the noise floor of the dataset is not a
   finding. Compute the floor before looking at the numbers.
4. **Model nondeterminism.** Five epochs per arm, variance reported. A single
   run is a pilot.
5. **Leakage.** Artifacts drawn from public code may be memorized. Every
   artifact is written for this dataset.
6. **Author bias.** The same person writes the artifacts, the `requires` groups,
   and the skill under test. Defects whose natural phrasing happens to match the
   skill's vocabulary will favor the skill arm. Mitigation: write `requires`
   groups before reading the skill text for that class, and record in
   `docs/CRITIQUE.md` that this bias is present and only partly controlled.
7. **Two skills is not eight.** No result here supports a claim about Dovetail
   as a whole.

## Milestones

Each milestone has an exit gate. Do not start the next one until the gate is met.

- **M1 Harness.** Arms, dataset loader, deterministic scorer, parameterized
  task, tests. Pilot dataset of 6 defects and 2 controls. Both arms run against
  a real model. Gate: every scorer failure mode has a passing test, and a
  mockllm run scores 0.0 with every sample recorded unparseable.
- **M2 Judge dataset.** Grown to 60 and 20, difficulty labeled, artifacts as
  files. Gate: a human review pass over every wrong-defect case from a full
  pilot run, with the alternates widened once and frozen.
- **M3 First result.** Placebo arm, five epochs, two model families. Gate:
  `docs/RESULTS.md` carries a dated entry naming the Dovetail revision, and
  `docs/CRITIQUE.md` carries a matching entry recording what it does not
  establish.
- **M4 Self-play.** Second suite, same harness, same scoring, new artifact
  class. Gate: the same as M3, plus an explicit statement of why a design flaw
  benchmark cannot be scored the same way twice if that turns out to be true.

## Non-goals

- Measuring the other six skills.
- Any claim about Dovetail as a whole.
- Model-graded scoring, at any milestone.
- Comparing Dovetail against other skill packs. That is a different experiment
  with different controls.

## Layout

```
assay/
  README.md
  SPEC.md
  IMPLEMENTATION_PROMPT.md
  AGENTS.md
  src/assay/
    arms.py          skill loading, revision capture, prompt assembly
    dataset.py       artifact loading and the requires contract
    scoring.py       deterministic scorer and the metrics
    tasks.py         parameterized Inspect tasks, one per suite
    report.py        arm comparison and the noise floor calculation
  artifacts/
    judge/           defect and control artifacts, one file each
    selfplay/
  tests/
  docs/
    RESULTS.md       dated runs, each naming its skill revision
    CRITIQUE.md      standing ledger of what the numbers do not prove
```

`docs/CRITIQUE.md` is not optional. Every published number gets a matching entry
recording what it does not establish.
