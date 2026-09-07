# assay

Measuring whether agent self-verification skills actually work.

## Status

Specification. No implementation exists in this repository yet. Two throwaway
sketches were run to confirm the harness shape is viable under Inspect AI
0.3.263; they are not the implementation and should not be imported.

## The claim under test

Dovetail (github.com/OpenCnid/dovetail) ships eight skills whose stated purpose
is to make Claude Code and Codex better at checking their own work. That claim
is currently unmeasured. No delta has ever been published for any of the eight.

Milestone 1 measures exactly one of them:

> Loading the `judge-composition` skill increases an agent's ability to
> correctly identify a planted defect in an artifact, without a matching
> increase in false positives on clean artifacts.

Self-play is Milestone 2 and reuses this harness. The remaining six skills are
out of scope until the method holds up.

## Method

A two-arm ablation over one dataset, one model, one response protocol. The only
thing that varies between arms is the presence of the skill text.

| Arm | System prompt |
| --- | --- |
| `baseline` | Bare review instruction plus the response protocol |
| `skill` | Skill text from a pinned Dovetail revision, then the same bare instruction and response protocol |

The reported result is the difference in defect recall between arms, alongside
the difference in false positive rate on controls. A skill that raises both
equally has not improved judgement, it has only made the model more suspicious.

### Placebo arm

The skill arm has a materially longer system prompt than baseline. Any measured
improvement could therefore be an artifact of prompt length or of the model
being told to think harder, rather than of the skill's actual content.

Milestone 3 adds a third arm, `placebo`, carrying a length-matched block of
generic review advice with none of judge-composition's structure. The honest
claim is the skill-over-placebo delta, not the skill-over-baseline delta.
Milestone 1 may report skill-over-baseline provided it is labeled as an upper
bound that has not controlled for prompt length.

## Skill loading contract

The skill text is never reproduced inside this repository. It is read at run
time from a Dovetail checkout so the benchmark always measures what actually
ships.

- Resolve `skills/judge-composition/SKILL.md`, falling back to `skill.md` then
  `README.md`.
- Fail loudly when the path is missing, when no skill file is found, or when
  the file is empty. A silently empty skill makes the two arms identical and
  reports a delta of zero for the wrong reason. This is the single most likely
  way for this benchmark to produce a confidently wrong result.
- Record the resolved Dovetail git revision in the eval log metadata. A result
  that cannot name the skill revision it measured is not a result.
- Pin the revision for any published run. A floating checkout is acceptable
  during development only.

## Dataset contract

A sample is an artifact plus the ground truth about what is wrong with it.

```
id            stable slug, e.g. "off-by-one-retry"
input         the artifact under review, verbatim
target        "DEFECT" or "CLEAN"
metadata.kind "defect" or "control"
metadata.planted   one sentence describing the planted defect, or "none"
metadata.requires  list of alternative-groups (see scoring)
```

Every defect sample carries exactly one planted defect. An artifact with two
plausible serious defects is ambiguous and belongs in the dataset only once the
scoring rule can express that.

Controls are load bearing, not filler. A judge that answers DEFECT on every
sample scores 100% on a defect-only set, and the benchmark measures nothing.
Target at least one control per three defect samples.

### Defect taxonomy

Milestone 1 pilot classes, one sample each, spanning obvious to subtle:

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

## Response protocol

Both arms end with the identical instruction. Deterministic scoring depends on
it being byte-identical across arms.

```
VERDICT: DEFECT | CLEAN
DEFECT: <one sentence naming the single most serious defect, or "none">
```

Report CLEAN only when there is no material defect. Style, naming, and missing
tests are explicitly not material defects; without that line, controls collect
false positives for reasons that have nothing to do with judgement.

## Scoring

Deterministic. No model grades the model. This is not a purity preference: a
model grader would be the largest uncontrolled variable in an experiment whose
entire output is a small delta.

- Unparseable verdict line: INCORRECT, recorded distinctly from a wrong answer.
  Protocol compliance is itself a measurement, and the skill arm's longer prompt
  may change it.
- Control sample: CORRECT if and only if the verdict is CLEAN. A DEFECT verdict
  is recorded as a false positive.
- Defect sample with a CLEAN verdict: INCORRECT, recorded as a miss.
- Defect sample with a DEFECT verdict: CORRECT if and only if every group in
  `requires` is satisfied. Each group is a set of alternative surface forms for
  one idea, so "off by one" and "one fewer retry" both count, while naming some
  other real bug does not. Recorded as a wrong-defect hit when it fails.

### Metrics

Report separately, never as one blended accuracy:

- Defect recall (defect samples correctly named)
- False positive rate (controls flagged DEFECT)
- Wrong-defect rate (flagged a defect, named the wrong one)
- Protocol compliance (parseable verdict lines)

## Threats to validity

Record these in the published result, not only here.

1. **Keyword brittleness.** A correct answer phrased outside every `requires`
   group scores as wrong. Mitigation: a human reads every wrong-defect case
   before any delta is published, and alternates are widened in the dataset
   rather than the scorer being loosened. Alternates may only be widened between
   published runs, never mid-run.
2. **Prompt length confound.** See the placebo arm above.
3. **Sample count.** Eight samples cannot support a claim about a skill. A delta
   below the noise floor of the sample count is not a finding. Set the target
   count before running, not after seeing the numbers.
4. **Model nondeterminism.** Run multiple epochs per arm and report variance.
   A single run per arm is a pilot, not a result.
5. **Leakage.** Defect artifacts drawn from public code may be memorized. Prefer
   artifacts written for this dataset.
6. **Single model.** A delta on one model is a claim about that model.

## Open decisions

To settle before Milestone 1 implementation begins:

- Target sample count and the defect-to-control ratio.
- Whether Milestone 1 ships the placebo arm or defers it to Milestone 3.
- Epochs per arm.
- Which models, and whether the published claim covers more than one.
- Pinned Dovetail revision for the first published run.
- Whether artifacts stay inline in the dataset module or move to `artifacts/`
  as files. Files scale better and diff more honestly.

## Milestones

- **M1** Harness plus the pilot dataset above. Both arms run. Result labeled a
  pilot, no delta published.
- **M2** Dataset grown to the agreed target count. Human review pass over every
  wrong-defect case.
- **M3** Placebo arm and multi-epoch runs. First publishable delta.
- **M4** Self-play, reusing this harness.

## Non-goals

- Measuring the other six skills.
- Any claim about Dovetail as a whole from a judge-composition result.
- Model-graded scoring, at any milestone.

## Layout

```
assay/
  SPEC.md
  IMPLEMENTATION_PROMPT.md
  AGENTS.md
  src/assay/
    arms.py          skill loading, prompt assembly, revision capture
    dataset.py       sample construction and the requires contract
    scoring.py       deterministic scorer and the four metrics
    tasks.py         the parameterized Inspect task
  artifacts/         defect and control artifacts, one file each
  docs/
    RESULTS.md       dated runs, each naming its skill revision
    CRITIQUE.md      standing ledger of what the numbers do not prove
```

`docs/CRITIQUE.md` is not optional. Every published number gets a matching entry
recording what it does not establish.
